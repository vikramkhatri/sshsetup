# sshsetup
Scripts to set  up password less SSH between multiple machines. Please fork this repository and use it to automate.

In order to set up password less SSH between machines, scripting brings consistency and less error prone compared to manual way of doing things. Setting up password less SSH is also a big task in many organizations and it keeps SAs very busy as many of them still do things by typing commands.

The main script `genSSH` gets the topology of the cluster from an environment variable `PS_TOPOLOGY_HOST`

## Prerequisites

The environment variables required are stored in `setenvvars` file.
```
#!/bin/bash

export SSH="/bin/ssh -q -o PreferredAuthentications=publickey \
            -o StrictHostKeyChecking=no"
export SCP="/bin/scp -q -o StrictHostKeyChecking=no"

export PS_LEADER=node01
export PUBLIC_DOMAIN=ibm.local
export PS_TOPOLOGY_HOST="PROXY:node01:192.168.142.101;MANAGEMENT:node01:192.168.142.101;MASTER:node02:192.168.142.102;WORKER:node03:192.168.142.103;WORKER:node04:192.168.142.104;WORKER:node05:192.168.142.105;WORKER:node06:192.168.142.106;WORKER:node07:192.168.142.107;WORKER:node08:192.168.142.108;WORKER:node09:192.168.142.109;WORKER:node10:192.168.142.110"

```
For example, if we are setting up password less SSH between 10 hosts, their topology is defined in `PS_TOPOLOGY_HOST` variable where each host entry is separated by a semicolon and colon is used to separate hosts attribute such as type, short name and IP address.

The script `genSSH` uses environment variable `PS_TOPOLOGY_HOST` to set up password less SSH between all hosts.

Complete the following prerequisites before running the script.

## Storing passwords so that the script can read it.

```
# mkdir -p /var/ps
# echo password | base64 > /var/ps/db2psc
# chmod 600 /var/ps/ps/db2psc
# chown db2psc.db2psc /var/ps/db2psc

# echo password | base64 > /var/ps/root
# chmod 600 /var/ps/ps/root
# chown root.root /var/ps/root

```
This is not the super cool way of doing things but works for me. I delete files from this directory after work is complete. This is just fool proof - meant to protect password from fools and not hackers. Fools may not know how to use base64 to decode.

## Copy `genSSH` and `exp` to /bin

If you want to allow your users to create their own password less SSH for their own user IDs for a given topology, consider copying `genSSH` and `exp` scripts to common bin folder such as `/bin` so that it is available to all. With the use of `setenvvars` in the home directory of the required user, the user has the flexibility to provide their own topology without changing the `genSSH` script.
```
# cp genSSH /bin
# cp exp /bin
# chmod 755 /bin/genSSH
# chmod 755 /bin/exp
```

## Set up `setenvvars` for individual users.

If you want to make `genSSH` available to all, consider copying `setenvvars` to their home directory.

```
cp setenvvars /home/db2psc/.setenvvars
```

The `genSSH` script reads this file from user's home directory and it is better to copy this as hidden.

## How it works?

The heart of the script `genSSH` is `exp` script that does the work for providing the password to ssh. This uses `expect` command so you must install it using `yum -y install expect`.

The motivation for the `exp` script is from this [link](https://stackoverflow.com/questions/4780893/use-expect-in-bash-script-to-provide-password-to-ssh-command).

I had to make some changes in the `exp` for it to run if password less SSH is already set up or when `genSSH` is invoked through a background process so that it should not fail in either case.

### Script `exp`
```
#!/usr/bin/expect

set timeout 20

set cmd [lrange $argv 1 end]
set password [lindex $argv 0]

eval spawn $cmd
expect {
"(yes/no)?"     {
        send -- "yes\r"
        exp_continue
        }
"*password:*" {
        send -- "$password\r"
        exp_continue
        }
eof
}
exit
```

Even if you are not using password less SSH, you could still use `exp` script to either run `ssh` or `scp` as the script `genSSH` shows. This is similar to `sshpass` command.

### Example of `exp` from `genSSH`

```
exp $USERPW ${SSH} $server rm -fr ${USERHM}/.ssh
var=$(printf '/bin/ssh-keygen -b 2048 -t rsa -N "" -f ~/.ssh/id_rsa')
exp $USERPW ${SSH} $server $var
exp $USERPW ${SCP} ${USERNM}@$server:${USERHM}/.ssh/id_rsa.pub /tmp/id_rsa.$server.pub
```

The first line shows the use of `exp` in deleting `.ssh` from all hosts. The second line builds the `ssh-keygen` string so that the whole string is stored within single quotes so that the `expect` command passes that string as it to the command being called. The 3rd line shows the usage of `exp` for `ssh` and the 4th line shows usage of `exp` for `scp`.

### Usage of `genSSH`

After meeting the prerequisites of 1. creating file in `/var/ps` for storing user password and copying `setenvvars` in the home directory of the end user, just run the `genSSH` and you are done.

```
$ whoami
db2psc

$ genSSH

# whoami
root

# genSSH
```

## Run as a background

I build my VMs from a single VMDK and use them using VMware Workstations. Basically, I use a script to set up hostname, network etc from the clone using the topology from environment variable `PS_TOPOLOGY_HOST`. As each VM is started, I run `setupSSH` from the leader machine through `/etc/rc.local` and it just waits for all machines to be up and then launches `genSSH` for `root` as well as `db2psc` user so that password less SSH is set up automatically. I had to use `eof` and `exit` in `exp` script for the background process to run properly - wasted several hours to figure this out.

The main logic in `setupSSH` is as follows.

```
while true
do
   live=1
   for server in ${sorted_hosts[@]};
   do
     live=$(islive $server)
     if [ $live -eq 1 ] ; then
        break;
     fi
   done
   if [ $live -eq 0 ] ; then
      echo ========================================
      echo "All are alive now. Sleep 3 then genSSH"
      echo ========================================
      sleep 3
      su - db2psc -c genSSH
      genSSH
      touch $SETUP_SSH_DONE
      sleep 2
      break
   else
      echo ========================================
      echo "Waiting for all to come alive"
      echo ========================================
      sleep 10
   fi
done
```

Please note that `genSSH` uses its own SSH alias whereas `setupSSH` uses SSH alias as defined in `setenvvars`. Both are different.

[Side note: `VMware Workstation` is the best virtualization software. Never use `Virtual Box` as that has performance problems and is very flaky. `VMware Workstation` is solid battle tested software. I run 10 VMs on my W540 laptop using VMware Workstation and run IBM Software - which you know are bulky and resource intensive. The Virtual Box does not stand up to VMware Workstation.]

## After password less SSH is done

* Delete encoded passwords from `/var/ps` folder.
* Disable password based authentication in `/etc/ssh/sshd_config`
* Protect your private keys as we did not use passphrase to encrypt the keys since we want to run commands using `ssh` on all machines and without being prompted for password or passphrase.
* You can be clever by still providing passphrase through `exp` to run the commands by accepting passphrase to `ssh` and `scp`. For that to occur, changes need to be done in the scripts. I run my machines in a protected environment so I do not care for using passphrase.

But we need better security - these are all band aid solutions to a broken security mechanism.

The ideal security is - No passwords, No credit card numbers and no bank account numbers. And, did I say no Certificate Authority? What will be there for hackers to steal? This can be done if tech industry has a desire. We are moving in that direction and will be there someday. The current way of security implementation is a job security for many and lots of revenues for Certificate Authority to issue certificates - a single source of truth - root certificate and what if, that root certificate is compromised through a court order or by force. We will never know.

## Pit falls of the genSSH script

Please be aware that this script will delete `.ssh` folder so this is destructive. If you have public and private keys in your `.ssh` folder, you might want to save them as this script will delete the `.ssh` directory.

We need password of the user (root or any user). I store them in /var/ps folder. This password must be same on all machines.

## License - Free for all. No need to even acknowledge.
