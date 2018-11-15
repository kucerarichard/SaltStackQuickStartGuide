# SaltStack Quick Start Guide
Quick Start Guide to SaltStack developed for use on Linux machines.  You need to modify for your machine naming patterns, and possibly the expect script depending on what state your machines are in right after being created (all with same temporary password for example). 

# INTRODUCTION


This guide provides a script and instructs you on how to launch a small SaltStack configuration on three of your Linux Academy lab machines.  The followup guides will contain simple orchestration labs,  some examples working with MySQL salt modules.   Related resources and notes are collected at the end of each guide.   

# GETTING STARTED


Please start 3 new Centos7 lab machines(use machine #1, #2, and #3) and then,  on machine #1:

1) log in,  sudo su
2) drop this script into machine #1 -- vim saltstack-init.sh and paste in the script.   Save it.
3) Then `chmod a+x saltstack-init.sh `
4) You should modify the SETPASSWD variable with your own password.  The complexity shown in the example password was enough to satisfy the ssh mechanism.  Remember that you have set the password on the machine #1 to this same complex password in order for the minion on machine #1 to install.  Avoid dollar signs,  the expect program cannot parse.
5) Run the script with command: `./saltstack-init.sh expect` (you need to run the script on machine #1, which becomes the salt master).
6) Wait... It seems to hang after running the expect,  don't worry it's still working,  takes a while...

```bash
#! /bin/bash
USERID=$(uname -a | awk '{ print $2 }' | cut -d '.' -f 1 | sed 's/.$//')
MASTERIP=$(ip addr show | grep '172.31' | awk '{ print $2 }' | cut -d '/' -f 1)
INITIALPASS='123456'
SETPASSWD='1z2x!3C#4V5BsdfCDb6n'
systemctl stop salt-master
systemctl stop salt-minion
yum-complete-transaction --cleanup-only
yum remove -y salt-ssh
yum remove -y salt-master
yum remove -y salt-minion
/bin/cp -f /dev/null /root/.ssh/known_hosts
curl -o bootstrap-salt.sh -L https://bootstrap.saltstack.com
sh bootstrap-salt.sh -M
yum install -y salt-ssh
yum install -y moreutils
yum install -y jq
yum install -y expect
# set up security for autosign.
# set up roster for salt-ssh init
cp /dev/null /etc/salt/autosign.conf
cp /dev/null /etc/salt/roster
for i in 1 2 3; do
echo ${USERID}${i}.mylabserver.com >> /etc/salt/autosign.conf
cat << ROSTER >> /etc/salt/roster
minion${i}:
  host: ${USERID}${i}.mylabserver.com
  user: user
  passwd: ${SETPASSWD}
  sudo: True
  tty: True
ROSTER
if [ "$i" -ne "1" ]
then
cat << MINIONEXPECT > /etc/salt/expect-minion${i}.tcl
#!/usr/bin/expect -f
spawn ssh ${USERID}${i}.mylabserver.com -l user
expect "?(yes/no)?"
send "yes\r"
expect "?assword:"
send "${INITIALPASS}\r"
expect "?assword:"
send "${INITIALPASS}\r"
expect "?assword:"
send "${SETPASSWD}\r"
expect "?assword:"
send "${SETPASSWD}\r"
interact
MINIONEXPECT
chmod a+x /etc/salt/expect-minion${i}.tcl
fi
done
# enable autosign.conf
sed -e 's/#autosign_file:/autosign_file:/' /etc/salt/master | sponge /etc/salt/master
# write some state files
mkdir -p /srv/salt/files/scripts
cat << STATE01 > /srv/salt/hello.sls
cmd_hello:
  cmd.run:
    - name: 'echo hello'
STATE01
cat << STATE02 > /srv/salt/sshd_off.sls
sshd_off:
  service.dead:
    - enable: False
STATE02
cat << STATE02B > /srv/salt/sshd_on.sls
sshd_on:
  service.running:
    - enable: True
STATE02B
cat << STATE03 > /srv/salt/minion.sls
minion_setup:
  cmd.script:
    - source: salt://files/scripts/salt-minion-init.sh
    - cwd: /tmp
    - user: root
#    - args: ${MASTERIP}
STATE03
cat << STATE04 > /srv/salt/emacs.sls
install_emacs:
  pkg.installed:
    - pkgs:
        - emacs
STATE04
cat << MINIONCFG > /srv/salt/files/scripts/salt-minion-init.sh
#! /bin/bash
systemctl stop salt-minion
yum-complete-transaction --cleanup-only
yum remove -y salt-minion
curl -o bootstrap-salt.sh -L https://bootstrap.saltstack.com
sh bootstrap-salt.sh
yum install -y moreutils
if grep --quiet salt /etc/hosts; then
sed -e 's/salt/abcdefg/g' /etc/hosts | sponge /etc/hosts
fi
# change master, edit /etc/hosts
echo "${MASTERIP}  salt" >> /etc/hosts
# edit salt-minion id
uname -a | awk '{ print \$2 }' > /etc/salt/minion_id
# change master, remove cached public key
rm -f /etc/salt/pki/minion/minion_master.pub
pip install pyinotify
systemctl restart salt-minion
MINIONCFG
# exit here to check/debug config
#exit 0
systemctl restart salt-master
if [[ $1 == "expect" ]]
then
for i in 2 3; do
/etc/salt/expect-minion${i}.tcl
done
fi
salt-ssh -i minion[123] state.sls minion

# Password set example, directly set hash in shadow
#PASSWD_HASH=$(salt-call --local shadow.gen_password '${SOMEPASSWD}' --out=json | jq '.local' | sed s/\"//g)
#salt '*' shadow.set_password user '${PASSWD_HASH}'
#salt '*' shadow.set_password root '${PASSWD_HASH}'
```

After about 6 minutes you will have a Salt master on machine #1, and authenticated minions on machines 1-3.
# WHAT IS SALT
For those who may have heard of SaltStack but are not yet familiar with it, SaltStack provides a very secure high speed godlike power for your bash shell and gives you access to a vast API normally reserved for more formal programming languages.
https://docs.saltstack.com/en/2015.8/ref/modules/all/index.html
It also gives you some very advanced ways to generate configuration through the use of templated constructs that can be thought of as your parameters populated at various times in the lifecycle of what salt terms “states” (similar to playbooks in Ansible, recipes in Chef). These states are executed by minions, which are powerful execution engines in their own right and can be invoked in bash without a master.

# HOW YOU CAN COMMAND, CONTROL AND ORCHESTRATE YOUR SYSTEMS WITH SALT
# Pillar Data
In Orchestrating with Salt, you can take a data-driven approach–say you have this complex system made up of many hosts and types of servers. You can “surface” select attributes that you want controlled or parameterized in a salt construct called a “pillar”, this is a custom yaml-based model–it’s up to you to design and organize the ergonomics of the yaml and the attribute outline how best you see fit.
For example, here is a pillar file that surfaces a bunch of attributes of interest from the configuration files of a JBoss cluster. This outline gets consumed by minions and builds clusters out of text file configuration, instead of special CLI, API or UI manipulation. JBoss still has very good support for regular file configuration, and text file configuration is SaltStack’s bread and butter(though there are dozens of APIs wrapped by Salt state and execution modules for you to use):

```yaml
clusters:
  testcluster01:
    bmanagement: 0.0.0.0
    enableinstance: True
    status: running
    maddress: 230.0.0.11
    balanceraddr: '*'
    balancerport: 80
    balancerallowfrom:
      - 123.34.56
      - 124.56.78
      - 123.34.23
    adgroupaddress: 224.0.1.106 
    adgroupport: 23364
    jspconfig:
      dev: 'true'
      checkinterval: 5
    launchhost: jboss-test1.zzzzz.zzzzz
    launchdir: /usr/local/testcluster01-deployments
    launchhandler: jboss-deploy.sls
    jbosshome: /usr/local/jboss-eap-6.4
    rsysloghost: 123.45.67.89
    cache:
       web:
         defaultstrategy: repl
#         defaultstrategy: dist
#         numcopies: 3
       hibernate:
         defaultstrategy: local-query 
#         defaultstrategy: replicated-cache 
    nodes:
      clusternode01:
        portoffset: 500
      clusternode02:
        portoffset: 600
      clusternode03:
        portoffset: 700
      clusternode04:
        portoffset: 800
```

This structure will provision 4 cluster nodes per host. If the pillar is sent to 4 hosts like so

`salt-run state.orchestrate orchestration.clusterbuilder`

16 jboss instances will be configured and join the cluster.
It is possible to add dynamic code in pillars. Here is an example of looping instead to create 20 JBoss instances per host to join the cluster spec’d in the pillar:

```jinja
{% for 10 .. 30 %}
      clusternode{{ i }}:
        portoffset: {{ i }}00
{% endfor %}
```

Minions receive from the master whatever pillar data they are entitled to see, based on their role, or name, ip, or any other of the numerous selectors(glob, regex, simple list) that salt can use to target minions.
# The Bus
Salt has a high-speed event bus, by default 0mq. SaltStack built an alternative transport called RAET (kind of a python riff on JGroups, the reliable UDP broadcast implementation for Java servers). But now for alternative transport they are advocating and developing Tornado instead, a python-based event-driven web server/communication framework.
To this bus are attached two features that allow you to simply access the bus–beacons (on the minions) and reactors (on the master). Beacons monitor for things (like using inotify functions for example) and send events when something happens. On the master reactors pick up those events and respond with whatever actions or orchestrated coordination needs to happen for that event.
In the lab in the next guide, we will run a salt command to watch the event bus, mainly to figure out what the tags are that go with beacon events, so we can write matching reactors to the events.

# WHAT TO DO WHEN THE SCRIPT FINISHES (about 6 minutes)
Test salt: in machine #1 type the following:

`salt '*' test.ping`
or
`salt \* test.ping`
You should see responses from 3 minions. The quotes are typically used when scripting salt commands and the backslash when calling directly from the command line.
Type:

`salt-key`

You should see a list detailing what keys the master has accepted. (the script takes security down one notch and “autosigns” all minions during the key exchange. After initialization you can deactivate autosigning in the master config (/etc/salt/master) and put the system back to maximum security–the default).
You can install emacs on one of your nodes:

`salt 'yourname3.mylabserver.com' state.sls emacs`

Emacs is one of the states embedded in the init script that got installed.
If your feeling brave you can turn off and disable sshd on all but your master node:

`salt 'yourname[23].*' state.sls sshd_off`

Note the globbing in the selector. Try rebooting the #2 machine and see if the minion comes back up :) Hopefully it does, then you can turn sshd back on again:

`salt 'yourname[23].*' state.sls sshd_on`

You may want to set root password on all your machines:

`salt-call --local shadow.gen_password 'newrootpassword' --out=json | jq '.local' | sed s/\"//g`

`$6$HopHHRQy$jclaXVI4unhM.....`

`salt '*' shadow.set_password root '$6$HopHHRQy$jclaXVI4unhM.....'`

# WHAT IS IN THE INIT SCRIPT
The script uses Expect to get into your machines to change the password.

```bash
#!/usr/bin/expect -f
spawn ssh ${USERID}${i}.mylabserver.com -l user
expect "?(yes/no)?"
send "yes\r"
expect "?assword:"
send "${INITIALPASS}\r"
expect "?assword:"
send "${INITIALPASS}\r"
expect "?assword:"
send "${SETPASSWD}\r"
expect "?assword:"
send "${SETPASSWD}\r"
interact
```

Then salt-ssh is used to get on the hosts and install the salt minion(it executes the ‘minion’ state) and related files in order for it to get authenticated and start talking to master.

`salt-ssh -i minion[123] state.sls minion`

The minion state:

```yaml
minion_setup:
  cmd.script:
    - source: salt://files/scripts/salt-minion-init.sh
    - cwd: /tmp
    - user: root
```

The embedded minion install script does the work on the minions (installed in the master salt filesystem and distributed by the salt state above). /srv/salt/files/scripts/salt-minion-init.sh is installed on the master by the Bash heredoc.

```bash
#! /bin/bash
systemctl stop salt-minion
yum-complete-transaction --cleanup-only
yum remove -y salt-minion
curl -o bootstrap-salt.sh -L https://bootstrap.saltstack.com
sh bootstrap-salt.sh
yum install -y moreutils
if grep --quiet salt /etc/hosts; then
sed -e 's/salt/abcdefg/g' /etc/hosts | sponge /etc/hosts
fi
# change master, edit /etc/hosts
echo "${MASTERIP}  salt" >> /etc/hosts
# edit salt-minion id
uname -a | awk '{ print \$2 }' > /etc/salt/minion_id
# change master, remove cached public key
rm -f /etc/salt/pki/minion/minion_master.pub
systemctl restart salt-minion
```

The salt-ssh roster was built and remains on master, so that salt-ssh can be used to do maintenance on the normal minion, like restart it, if there’s a problem. Another common maintenance task is to delete the cached public key from the master on the minion, if the master has moved or changed IPs.
SaltStack publishes a bootstrap script that makes salt handle installation across various flavors. Let that do the work. Without the -M it just installs the minion.

```bash
curl -o bootstrap-salt.sh -L https://bootstrap.saltstack.com
sh bootstrap-salt.sh
```

There are some embedded states. This one installs emacs:

```yaml
install_emacs:
  pkg.installed:
    - pkgs:
        - emacs
```

The name of the state is simply established by the name of the state file, in this case ‘emacs.sls’ was installed into the master filesystem(/srv/salt) by the Bash heredoc. The salt master filesystem is declared in master config /etc/salt/master.
The task of assigning master to the minion is done by setting the host for salt (default hostname for master):

```bash
# change master, edit /etc/hosts
echo "${MASTERIP}  salt" >> /etc/hosts
Note: there is a way to assign the master host by putting that attribute in a separate include file and placing in /etc/salt/minion.d. Probably a good idea, but for now we edit the hosts file, since “salt” is the default name of the master out of the box.
There is the task of overriding the minion ID:

# edit salt-minion id
uname -a | awk '{ print \$2 }' > /etc/salt/minion_id
The ID grabbed by the salt minion install (the one generated for Linux Academy machines) was too cumbersome to use. The second field of ‘uname -a’ provides a better name.
Finally, the minion must reset the master:

# change master, remove cached public key
rm -f /etc/salt/pki/minion/minion_master.pub
```

This is done in the case of multiple reinstalls (without expect), it just wipes out the public key of the master stored on the minion so that it gets re-exchanged. If you ever move master you have to do this.

# THE END
I hope you had success with this guide and continue to delve into SaltStack.   See you in the next guide!  

# SOURCES / RESOURCES


## General

https://docs.saltstack.com/en/latest/
https://github.com/hbokh/awesome-saltstack
http://www.yet.org/2016/09/salt/
https://arnoldbechtoldt.com/blog/saltstack-current-challenges
http://www.saltstat.es/

## Resources for the saltstack-init.sh script

http://stackoverflow.com/questions/15432275/running-command-with-expect
http://stackoverflow.com/questions/2823007/ssh-login-with-expect1-how-to-exit-expect-and-remain-in-ssh
http://bencane.com/2016/07/19/using-salt-ssh-to-install-salt/
https://arstechnica.com/civis//viewtopic.php?f=16&t=1171622
https://docs.saltstack.com/en/latest/ref/modules/all/salt.modules.shadow.html#salt.modules.shadow.gen_password
http://serverfault.com/questions/485031/what-password-hash-algorithm-does-each-version-of-centos-use-by-default
http://stackoverflow.com/questions/19640829/how-can-i-execute-multiple-commands-using-salt-stack
https://docs.saltstack.com/en/latest/ref/states/all/salt.states.cmd.html#salt.states.cmd.script
https://docs.saltstack.com/en/latest/topics/ssh/roster.html#ssh-roster
https://docs.saltstack.com/en/latest/topics/ssh/
https://docs.saltstack.com/en/latest/ref/modules/all/salt.modules.shadow.html#salt.modules.shadow.set_password
https://docs.saltstack.com/en/latest/ref/states/all/salt.states.service.html#salt.states.service.dead
http://stackoverflow.com/questions/2953081/how-can-i-write-a-here-doc-to-a-file-in-bash-script
http://ss64.com/bash/
http://unix.stackexchange.com/questions/119269/how-to-get-ip-address-using-shell-script
https://en.wikipedia.org/wiki/Cut_(Unix)
http://stackoverflow.com/questions/3323809/trim-last-3-characters-of-a-line-without-using-sed-or-perl-etc
https://stedolan.github.io/jq/
https://stedolan.github.io/jq/manual/
https://jqplay.org/

# ORCHESTRATION LAB

This is a simple orchestration lab. Related resources and notes are collected at the end of the guide.

Let’s configure and orchestrate rsyslog and learn about the following:

Jinja templates
Jinja macros
Minion Beacons
Master Reactors

You should have lab machines #1 (master), #2 (minion), and #3 (minion) setup with salt. 

## Objective

On each minion, create a file, /tmp/salt-notes.txt, and use SaltStack to monitor any changes to that file. When the file is modified, push the last line in the file, on only that minion, to the rsyslog service on master, where it should be filtered into a special log file /var/log/notes/handy-salt.log.

# GETTING STARTED


## Setup

All of the source files below are placed in the master(machine #1) salt filesystem /srv/salt.
Place the following library in /srv/salt/lib/loggerlib.sls

```
{% macro loggerbundle(path2logfile,tag) -%}
/opt/logger.local/{{ tag }}-logger.sh:
    file.managed:
        - makedirs: True
        - source: salt://files/rsyslog/logger.sh.jinja
        - user: root
        - group: root
        - mode: 755
        - template: jinja
        - defaults:
              path2logfile: {{ path2logfile }}
              tag: {{ tag }}
{%- endmacro %}

Place the following 3 files in a template directory for rsyslog
/srv/salt/files/rsyslog:

rsyslog-listener.conf.jinja
$ModLoad imudp
$UDPServerRun 514
logsplitter.conf.jinja
if $syslogtag contains 'handy-salt' then /var/log/notes/handy-salt.log
logger.sh.jinja
{% set masterip = salt['cmd.run']('grep salt /etc/hosts | awk "{ print \$1; }"') -%}
/usr/bin/tail -1 {{ path2logfile }} | /usr/bin/logger -n {{ masterip }} -P 514 -t {{ tag }} 

Place the following state in /srv/salt/notelog.sls
{% from 'lib/loggerlib.sls' import loggerbundle with context %}
{{ loggerbundle('/tmp/salt-notes.txt','handy-salt') }}

/tmp/salt-notes.txt:
   file.touch

Place the following beacon in /srv/salt/files/minion.d/beacon.conf.jinja
beacons:
  inotify:
     /tmp/salt-notes.txt:
        mask:
          - modify
     disable_during_state_run: True

Add these 2 state files to /srv/salt:
1) rsyslog.sls
/etc/rsyslog.d/rsyslog-splitter.conf:
  file.managed:
    - source: salt://files/rsyslog/logsplitter.conf.jinja
    - template: jinja

/etc/rsyslog.d/rsyslog-listener.conf:
  file.managed:
    - source: salt://files/rsyslog/rsyslog-listener.conf.jinja
    - template: jinja
2) beacons.sls
/etc/salt/minion.d/beacon.conf:
  file.managed:
    - source: salt://files/minion.d/beacon.conf.jinja
    - template: jinja
    - makedirs: True
```

Beacons are a way to make the salt event bus available through simple configuration to translate system events into salt events, which can be reacted to by the Salt master in Reactors. Minions are instrumented with beacon modules, there are a lot these days:

https://docs.saltstack.com/en/latest/ref/beacons/all/index.html


## Let’s run this stuff

1) Apply the rsyslog state to master (your machine #1)
`salt yourusername1.* state.sls rsyslog`

2) Apply the notelog state to all
`salt \* state.sls notelog`

3) Restart rsyslog
- confirm it’s there 
`salt \* service.get_all | grep rsyslog `
- restart it 
`salt \* service.restart rsyslog`

4) Test the notelog
`salt \* cmd.run 'echo 56789 >> /tmp/salt-notes.txt'`
`salt \* cmd.run /opt/logger.local/handy-salt-logger.sh`
`tail /var/log/notes/handy-salt.log`

5) Deploy the beacon, and restart minions to pick up the beacon (use salt-ssh because the minion can’t restart itself without failing to return something from the operation, so there is a timeout)
`salt \* state.sls beacons`
`salt-ssh minion[123] service.restart salt-minion`

6) Configure the reactor quick and dirty  :)  You need to watch the event bus to figure out what the tags are.  
On master:
`salt-run state.event pretty=true`
If you modify the salt-notes.txt file,  the event bus will show that you need something that looks like this:
‘salt/beacon/username2.mylabserver.com/inotify//tmp/salt-notes.txt’
You just need a few more files to tie together individual minions to their individual reactors – run this quick and dirty script to generate reactor configuration and sls files directly into the master configuration.  See how the alternative format --out=text on the salt command simplifies the output to make it easier to grab a list of available minions. Crude but effective.

generate-reactor-code.sh:
```bash
#!/bin/bash
#
# Quick and dirty script to generate reactor conf and state files
# Run this on master.
# Reactor SLS is a little different.  See
# https://docs.saltstack.com/en/getstarted/event/reactor.html
#
echo "reactor:" > /etc/salt/master.d/reactor.conf
for i in `salt \* test.ping --out=text | awk '{ print $1; }' | sed -e 's/.$//'`
do
cat << REACT01 >> /etc/salt/master.d/reactor.conf
  - 'salt/beacon/${i}/inotify//tmp/salt-notes.txt':
    - salt://reactor-${i}.sls
REACT01
cat << REACT02 > /srv/salt/reactor-${i}.sls
send ${i} note to syslog:
  local.cmd.run:
    - tgt: '${i}'
    - arg:
      - /opt/logger.local/handy-salt-logger.sh
REACT02
done
```

7) Test the reactor config.  Restart master in debug  just to make sure it parses the new file. It should show no errors. Verify, ^C to stop the debug master, then restart the service.

`systemctl stop salt-master`
`salt-master -l debug`
...
`^C`
`systemctl start salt-master`
8) Finally
Pick one of the minions and either edit or echo >> some text at the bottom of /tmp/salt-notes.txt
On master check the log file /var/log/notes/handy-salt.log, you should see your text logged.
vi will generate two events for inotify modify, so you will see duplicate log entries
echo >> generates only one event for inotify modify
If you want to see the beacon/reactor in action, start both the master and a minion in debug mode (-l debug) in two separate terminals. On a third terminal on master listen to the event log(step 6 above). Then use a fourth terminal to log into the minon and edit the /tmp/salt-notes.txt file.

# THE END
There are many more aspects to SaltStack not covered here, such as requisites, salt-cloud, the actual orchestrate runner, etc. that might be covered on future posts.   I hope you had success with this guide and continue to delve into SaltStack.
SOURCES / RESOURCES


Debugging and Tweaking the Install
IRC
http://irclog.perlgeek.de/salt/ (archives)
http://webchat.freenode.net/?channels=salt (join chat)
This guide was almost derailed by an inoperable beacon system. Fortunately because we have IRC logs for Salt it only cost me a couple hours sleep.
Shutting down the salt-minion and starting in debug is easy enough:

`systemctl stop salt-minion`
`salt-minion -l debug`
What this revealed is a failure to load the beacon file.
See this IRC log for relevant conversation, ending with the fact that if you pip install the dependency the problem is fixed:
https://irclog.perlgeek.de/salt/2016-04-06#i_12297672

pip install pyinotify

You’ll see that this line has been added to the minion installer in saltstack-init.sh in the SaltStack Quick Start Guide.
There are other options, such as specifying which version that bootstrap-salt.sh should load, thereby avoiding the newer dependency. What likely happened is that features were added to the inotify part of the beacon system, creating a requirement for pyinotify update and the Centos 7 yum repo was no longer adequate to run it. If you know package systems and python well, you could possibly update your yum repos to load an updated pyinotify. Fortunately the alternative package system for python pip delivered what was required.

https://pypi.python.org/pypi/pyinotify
https://docs.saltstack.com/en/latest/ref/beacons/all/salt.beacons.inotify.html
depends:    pyinotify Python module >= 0.9.5

Beacon/Reactor documentation is online:
https://docs.saltstack.com/en/latest/topics/reactor/
https://docs.saltstack.com/en/latest/topics/event/index.html
https://docs.saltstack.com/en/latest/topics/beacons/index.html
https://docs.saltstack.com/en/getstarted/event/reactor.html
Advanced orchestration 
https://github.com/kucerarichard/SaltedJBoss

Resources for rsyslog
http://download.rsyslog.com/design.pdf
https://media.readthedocs.org/pdf/rsyslog/latest/rsyslog.pdf
http://unix.stackexchange.com/questions/128156/remote-syslog-command-line-client
http://www.rsyslog.com/sending-messages-to-a-remote-syslog-server/
http://kwlug.org/sites/kwlug.org/files/2009-08-10-syslog-servers.pdf
http://lists.adiscon.net/pipermail/rsyslog/2013-August/033421.html

https://docs.saltstack.com/en/latest/topics/beacons/index.html

