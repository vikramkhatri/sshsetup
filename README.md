# Set up password less SSH between machines

## Prerequisites

Install `sshpass` in all machines.

```
yum -y install sshpass
```

## Prepare inventory file

The inventory file contains the IP address, FQDN, Short name and the password of the user

Example:

```
192.168.142.173 node01.zinox.local node01 SuperComplexPasswordAHackerCantGuess
192.168.142.143 node02.zinox.local node02 SuperComplexPasswordAHackerCantGuess
192.168.142.253 node03.zinox.local node03 SuperComplexPasswordAHackerCantGuess
192.168.142.227 node04.zinox.local node04 SuperComplexPasswordAHackerCantGuess
192.168.142.178 node05.zinox.local node05 SuperComplexPasswordAHackerCantGuess
192.168.142.152 node06.zinox.local node06 SuperComplexPasswordAHackerCantGuess
192.168.142.239 node07.zinox.local node07 SuperComplexPasswordAHackerCantGuess
192.168.142.138 node08.zinox.local node08 SuperComplexPasswordAHackerCantGuess
192.168.142.203 node-gui.zinox.local node-gui SuperComplexPasswordAHackerCantGuess
10.45.176.104 node01-g.zinox.local node01-g SuperComplexPasswordAHackerCantGuess
10.45.176.44  node02-g.zinox.local node02-g SuperComplexPasswordAHackerCantGuess
10.45.176.119 node03-g.zinox.local node03-g SuperComplexPasswordAHackerCantGuess
10.45.176.88  node04-g.zinox.local node04-g SuperComplexPasswordAHackerCantGuess
10.45.176.16  node05-g.zinox.local node05-g SuperComplexPasswordAHackerCantGuess
10.45.176.26  node06-g.zinox.local node06-g SuperComplexPasswordAHackerCantGuess
10.45.176.54  node07-g.zinox.local node07-g SuperComplexPasswordAHackerCantGuess
10.45.176.40  node08-g.zinox.local node08-g SuperComplexPasswordAHackerCantGuess
10.45.176.74  node-gui-g.zinox.local node-gui-g SuperComplexPasswordAHackerCantGuess
```

## Run genSSH

Run genSSH with inventory file to exchange keys from each host to the other

```
# ./genSSH ip.txt
```

## Run testSSH

Run testSSH with same inventory file to do the round robin tests to make sure that the ssh with public / private keys works.


```
# ./testSSH ip.txt
```
