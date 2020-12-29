# Introduction

Sanboxing use kernel namespaces to isolate processus, it wrap global systems ressources and make their appeir that its own the global ressourses
It allow to seperate group of processus that they do not see each others
It create an environment close to a chroot, with seperate system ressources this allow to have a better security.


There is 6 differents namespace in the linux kernel.
* Mount 
* UTS
* IPC
* PID
* Network
* User

## Firejail

Firejail is a SUID program allowing to run processus within its own "jail".
Seperate from the user environment for better security, it also use secomp-bpf for graphical program.
Its originaly used to reduce the risk of security for untrusted processus.


### Use

```sh
firejail --private <program>
```

### Example

```sh
firejail --private termite
```

output: 

```sh
USER         PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
xxx            1  0.0  0.0   4788  2840 pts/1    S+   00:31   0:00 firejail termite
xxx            3  1.1  0.0 410616 32272 pts/1    Sl+  00:31   0:00 termite
xxx            7  0.5  0.0   6872  5288 pts/2    Ss   00:31   0:00 /bin/zsh
xxx           51  0.0  0.0   6624  2832 pts/2    R+   00:31   0:00 ps aux
```

Sources:
* https://lwn.net/Articles/531114/
* https://wiki.archlinux.org/index.php/Firejail

