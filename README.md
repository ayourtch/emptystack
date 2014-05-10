emptystack
==========

I tend to launch qemu/kvm quite a lot on my lab hosts. Way too much. So much it becomes boring. Need to automate.

Openstack ? Levels and levels of indirection, tons of options, an entire bag of new semantics... 
No, I need something more stupid.

Docker ? this looks a bit simpler, but still a bit too much of work.

Vagrant ? this seems relatively close. But unless I misunderstood, it wants VirtualBox, while I wanted kvm. Yes, there is a plugin to it, but this again starts to get complicated...

Thus, vmess was born.

It is a completely unscalable way to manage VMs. Don't use it in production. It's just an experiment. It's not finished.

Anyway, enough whining, let's see what's the big deal about.


This is the overview of the functions, given by "vmess help":

```
vmess: VM Empty Stack Supervisor: A wrapper around bare-bones KVM virtual machines using linux bridged networking.

Usage:

vmess top
      Run a "top" command, filtering just the KVM instances

vmess net 
      Various operations on the bridge, as a whole

vmess <vmname> ...
      Various operations on VMs. VM should be in a form [a-z]+[0-9]

vmess <vmname> make
      create a directory for the VM

vmess <vmname> smp <N>
      Set the "-smp" parameter of KVM to <N> and store it inside the VM's dir.

vmess <vmname> ram <size>
      Set the size of RAM for the VM and store it inside the VM's dir.


vmess <vmname> hda <size>
vmess <vmname> hdb <size>
      Create a sparse file for the first and second disk.
      Note that because the file is created with a "seek" option of dd, 
      the subsequent invocations of the command do not delete the file,
      they only overwrite the very last byte to be zero.

vmess <vmname> cdrom <path-to-iso>
      Create a symlink to the cdrom to be used by the VM.

vmess <vmname> ifadd <N> [ <bridge> <bridge> ... ]
      Add N virtual NICs to the VM, and optionally place them into
      bridged segments. Create the segments if necessary. 
      You can use "-" as a denomination of a bridge to indicate
      you do now want this interface to be connected anywhere.
      NB: If the VM has the existing network cards, they are removed first.

vmess <vmname> ifdel
      Delete all of the VM's NICs. Called by ifadd, if there are existing NICs.

vmess <vmname> connect <ifnum> <bridge>
      Connect Nth interface (<ifnum>, starts from 0) to a specified <bridge>,
      disconnecting it from the current bridge if necessary. Somewhat redundant
      with "ifadd", but can be useful for manipulating the interfaces one by one.

vmess <vmname> ifstat
      Display the VM NICs' current connections in the syntax of a "ifadd" 
      command above.

vmess <vmname> start
      Start the VM in the background.

vmess <vmname> stop
      Stop (kill) the VM.

vmess <vmname> status
      Show the status of the VM - whether it is running or not.

vmess <vmname> info
      Show the various information about the VM, including the current full command
      line used to launch KVM.

vmess <vmname> vnc
      Start a socat instance relaying the main machine's display to the VM's vnc console.
      This socat instance serves a single connection attempt from the operator.

```


Let's toy with a vm:

```
ayourtch@mcpico:~$ vmess newvm
You need to create directory /home/ayourtch/vms/newvm0 if are serious about that VM - use vmess make
ayourtch@mcpico:~$ 
```

Uh okay, let's do it.

```
ayourtch@mcpico:~$ vmess newvm make
You need to create directory /home/ayourtch/vms/newvm0 if are serious about that VM - use vmess make
Oh, I see you are serious. Okay, I will proceed.
ayourtch@mcpico:~$ 
```

So, what now ? This is just an empty directory, after all:

```
ayourtch@mcpico:~$ ls -al ~/vms/newvm0/
total 8
drwxrwxr-x  2 ayourtch ayourtch 4096 May  6 22:53 .
drwxrwxr-x 10 ayourtch ayourtch 4096 May  6 22:53 ..
ayourtch@mcpico:~$ 
ayourtch@mcpico:~$ vmess newvm info
Total number of interfaces: 0
Command for this VM
  kvm  -vnc 127.0.0.1:2811 -daemonize
VM not running
ayourtch@mcpico:~$ 
```

Still not a whole lot. Let's add some interfaces....

```
ayourtch@mcpico:~$ vmess newvm ifadd 3 bridge1 bridge2 bridge3
Set 'newvm0e0' persistent and owned by uid 1000
Making a VLAN: bridge1
preparing a TAP interface newvm0e0
Set 'newvm0e0' persistent and owned by uid 1000
Set 'newvm0e1' persistent and owned by uid 1000
Making a VLAN: bridge2
preparing a TAP interface newvm0e1
Set 'newvm0e1' persistent and owned by uid 1000
Set 'newvm0e2' persistent and owned by uid 1000
Making a VLAN: bridge3
preparing a TAP interface newvm0e2
Set 'newvm0e2' persistent and owned by uid 1000
ayourtch@mcpico:~$ 
```

Now there was some action... Let's see what is the result:

```
ayourtch@mcpico:~$ brctl show
bridge name	bridge id		STP enabled	interfaces
bridge1		8000.9e351783e6ca	no		newvm0e0
bridge2		8000.daa0a9a49968	no		newvm0e1
bridge3		8000.9a0acaa90f4c	no		newvm0e2
ayourtch@mcpico:~$
```

This is not too bad ?

let's check the info:

```
ayourtch@mcpico:~$ vmess newvm info
Total number of interfaces: 3
Command for this VM
  kvm  -net tap,vlan=0,ifname=newvm0e0,script=/bin/true -net nic,vlan=0,macaddr=02:11:9b:af:00:00 -net tap,vlan=1,ifname=newvm0e1,script=/bin/true -net nic,vlan=1,macaddr=02:11:9b:af:00:01 -net tap,vlan=2,ifname=newvm0e2,script=/bin/true -net nic,vlan=2,macaddr=02:11:9b:af:00:02 -vnc 127.0.0.1:2811 -daemonize
VM not running
ayourtch@mcpico:~$ 
```

The command line is growing... 

But still nothing in the directory:

```
ayourtch@mcpico:~$ ls -al ~/vms/newvm0/
total 8
drwxrwxr-x  2 ayourtch ayourtch 4096 May  6 22:53 .
drwxrwxr-x 10 ayourtch ayourtch 4096 May  6 22:53 ..
ayourtch@mcpico:~$
```

The script picks up the interfaces named according to the convention, and associates them with the VM.

Looks interesting ?

We can as easily wipe the interfaces, and this will shrink the command line back:

```
ayourtch@mcpico:~$ vmess newvm ifdel
Set 'newvm0e0' nonpersistent
Set 'newvm0e1' nonpersistent
Set 'newvm0e2' nonpersistent
ayourtch@mcpico:~$ vmess newvm info
Total number of interfaces: 0
Command for this VM
  kvm  -vnc 127.0.0.1:2811 -daemonize
VM not running
ayourtch@mcpico:~$ 
```

Let's add back an interface and put it in vlan100 - this is an internet-connected segment.

```
ayourtch@mcpico:~$ vmess newvm ifadd 1 vlan100
Set 'newvm0e0' persistent and owned by uid 1000
Making a VLAN: vlan100
device vlan100 already exists; can't create bridge with the same name
preparing a TAP interface newvm0e0
Set 'newvm0e0' persistent and owned by uid 1000
ayourtch@mcpico:~$ 
```

Before we go further, we should probably create a hard disk for this VM. Let's take 4G:

```
ayourtch@mcpico:~$ vmess newvm hda 8G
1+0 records in
1+0 records out
1 byte (1 B) copied, 4.3652e-05 s, 22.9 kB/s
ayourtch@mcpico:~$ 
```

And assign a bit more RAM than default:

```
ayourtch@mcpico:~$ vmess newvm ram 2G
ayourtch@mcpico:~$ 
```

Now, let's associate the ubuntu boot ISO with this VM:

```
ayourtch@mcpico:~$ vmess newvm cdrom ~/iso/ubuntu-14.04-server-amd64.iso 
ayourtch@mcpico:~$ 
```

Let's see what we have in the directory:

```
ayourtch@mcpico:~$ ls -alh ~/vms/newvm0/
total 16K
drwxrwxr-x  2 ayourtch ayourtch 4.0K May  6 23:01 .
drwxrwxr-x 10 ayourtch ayourtch 4.0K May  6 22:53 ..
lrwxrwxrwx  1 ayourtch ayourtch   48 May  6 23:01 newvm0-cdrom.iso -> /home/ayourtch/iso/ubuntu-14.04-server-amd64.iso
-rw-rw-r--  1 ayourtch ayourtch 8.1G May  6 23:00 newvm0-hda.img
-rw-rw-r--  1 ayourtch ayourtch    3 May  6 23:00 RAM
ayourtch@mcpico:~$ 
```

just the hard disk image, a symlink to the ISO, and a small file which contains the amount of RAM to be passed to kvm:

```
ayourtch@mcpico:~$ cat ~/vms/newvm0/RAM 
2G
ayourtch@mcpico:~$ 
```

Now, let's check the info:

```
ayourtch@mcpico:~$ vmess newvm info
Total number of interfaces: 1
Command for this VM
  kvm  -net tap,vlan=0,ifname=newvm0e0,script=/bin/true -net nic,vlan=0,macaddr=02:11:9b:af:00:00 -vnc 127.0.0.1:2811 -hda /home/ayourtch/vms/newvm0/newvm0-hda.img -cdrom /home/ayourtch/vms/newvm0/newvm0-cdrom.iso -m 2G -daemonize
VM not running
ayourtch@mcpico:~$ 
```

And now we can start the VM:

```
ayourtch@mcpico:~$ vmess newvm start
Starting VM: kvm  -net tap,vlan=0,ifname=newvm0e0,script=/bin/true -net nic,vlan=0,macaddr=02:11:9b:af:00:00 -vnc 127.0.0.1:2811 -hda /home/ayourtch/vms/newvm0/newvm0-hda.img -cdrom /home/ayourtch/vms/newvm0/newvm0-cdrom.iso -m 2G -daemonize
ayourtch@mcpico:~$ vmess
List of KVMs (with PIDs as in ps-ef):
ayourtch  5748     1 20 23:03 ?        00:00:01 -vnc :2811 -hda newvm0/newvm0-hda.img -cdrom newvm0/newvm0-cdrom.iso -m 2G -daemonize
ayourtch@mcpico:~$ 
ayourtch@mcpico:~$ vmess newvm
VM is running, PID: 5748
ayourtch@mcpico:~$ 
```

Notice that the VNC is running on loopback interface, so we can not really connect to this VM from anywhere else.

Not to worry, if you have socat installed, you can put in a temporary relay for a single connection:

```
ayourtch@mcpico:~$ vmess newvm vnc
Please VNC to 5900 to connect to this VM console
ayourtch@mcpico:~$ ps -ef | grep socat
ayourtch  5846     1  0 23:05 pts/0    00:00:00 socat TCP-LISTEN:5900,reuseaddr TCP:localhost:8711
ayourtch  5854  9919  0 23:05 pts/0    00:00:00 grep --color=auto socat
ayourtch@mcpico:~$ 
```

The socat services only a single connection, so this achieves a pretty good security without the extra complexity -
since the VNC server is not listening, you do not need to give it a password!

After you're done with running the VM, you can stop it:

```
ayourtch@mcpico:~$ vmess newvm stop
Stopping VM PID 5748
ayourtch@mcpico:~$ 
```

And clean up the network interfaces:

```
ayourtch@mcpico:~$ vmess newvm ifdel
Set 'newvm0e0' nonpersistent
ayourtch@mcpico:~$ 
```

And now, let's clean it up completely:

```
ayourtch@mcpico:~$ rm -rf ~/vms/testvm0/
ayourtch@mcpico:~$
```

Done ! No traces of the VM on this host !




