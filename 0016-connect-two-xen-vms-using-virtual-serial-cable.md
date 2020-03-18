# Xen - Connect two VMs via Virtual Serial Port

I have recently been working on debugging The Windows kernel.  

For the version of Windows I am using (7 professional / 32-bit), the easiest way to debug the kernel is via serial port.

In VirtualBox this is easy, as VirtualBox provides robust serial port options that allow the user to specify a unix socket to use as a virtual serial cable.  If the unix socket doesn't exist before the VM is booted VirtualBox, when booting the server VM, will create the unix socket on the filesystem.  The client VM can then connect to the unix socket by specifying the same path in the VirtualBox serial port settings.

However, for my purposes it became necessary to use Xen as the hypervisor.  I started by installing and configuring Xen on an Ubuntu server, then spinning up two Windows HVM guest VMs.  

At this point I looked far and wide for documentation on how to connect the two HVM guests via a virtual serial port in Xen.  I found none (hence this blog gist).

Eventually, through experimentation, I figured out a way to connect the two VMs using socat.  

The socat command I used is:

```bash
socat UNIX-LISTEN:/tmp/windows-debugger UNIX-LISTEN:/tmp/windows-debuggee
```

Then in the debugger HVM guest configuration file (.cfg), I added the following line:

```
serial = 'unix:/tmp/windows-debugger'
```

And in the debuggee HVM guest configuration file (.cfg), I added a similar line:

```
serial = 'unix:/tmp/windows-debuggee'
```

I then start the debugger VM in dom0 using `xl create` and connect via RDP.  Then, I start WinDbg on the debugger VM and fire up a kernel listener.  After the listener is started, I launch the debuggee VM (which is configured for kernel debugging), and the serial connection will be established - with output on the debugger WinDbg instance.

It is worth noting that once either VM shuts down or otherwise disconnects from the socket, it will be necessary to shutdown the other VM and then shutdown the socat process.  In other words, there is no way to reuse the socat process.  It must be recreated on every VM boot for either guest.


[Back](https://nstarke.github.io/)
