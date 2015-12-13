
# kptrace

Uses `lldb` to dump the stack trace from an OS X kernel panic.

## Installation

Just take the script `kptrace` from this repo and place it somewhere in your`$PATH`.

`kptrace` works with Python 3.x. 
Typically when you install Python 3.x for OS X, the installer sets up your `$PATH` so that the command `/usr/bin/env python3` resolves to the Python 3.x interpreter executable.
If running `/usr/bin/env python3` in your terminal doesn't start Python 3.x, open the `kptrace` script and change the `#!` in the preamble to point to your Python 3.x interpreter.

## Usage

`kptrace` requires OS X kernel symbols, available from Apple in the form of a Kernel Development Kit (KDK).
You can download the appropriate KDK for your OS X build from the [Apple Developer downloads page](https://developer.apple.com/downloads).

Once you have installed the appropriate KDK for the crashing build, you can use `kptrace` as follows:

    $ kptrace [-report <path to report>] [-kdk <kdk dir] [-kext <kext>]

All arguments are optional, and behave as follows:

* `-report`, if specified, points to the kernel panic report you want to parse.
  This is the report which you are prompted to send to Apple one boot after the kernel panic.
  If you omit this argument, `kptrace` finds the most recent `.panic` report in `/Library/Logs/DiagnosticReports`.

* `-kdk`, if specified, points to the kernel development kit to load kernel symbols from.
  If not specified, `kptrace` finds the version number in the panic and finds the matching KDK in `/Library/Developer/KDKs`.

* `-kext` specifies an additional .kext file to include in the report.
  You need to add this if the crash contains your own third-party kexts.
  You can specify `-kext` multiple times for different kexts.

## How it Works

A nice step-by-step for obtaining a kernel panic backtrace is availalbe [on the darwin-kernel list here](http://lists.apple.com/archives/darwin-kernel/2014/Jan/msg00011.html).
This script basically emulates the same behavior by generating an lldb command script and feeding it into lldb.

The basic steps in that script are:

* Call `target create` to load the kernel into the debugger
* Parse the kexts in the backtrace and `target modules add` each.
* Parse the kernel slide from the crash report, and `target modules load` the kernel at the correct address.
* Parse the `__TEXT` offset for each kext in the backtrace, and `target modules load` each kext at the right address.
* Parse the backtrace return addresses from the report, and `image lookup -a` each return address

The last step produces a full list of code locations for return addresses on the stack.
Finally, we pass that through a couple filters to produce something nice like the following:

```
  - kernel`panic + 226 at debug.c:402
  - UplinkOSXDevice`UplinkAudioDevice::taggedRetain(void const*) const + 58 at UplinkAudioDevice.cpp:77
  - kernel`OSArray::setObject(unsigned int, OSMetaClassBase const*) + 269 at OSArray.cpp:254
  - kernel`IORegistryEntry::makeLink(IORegistryEntry*, unsigned int, IORegistryPlane const*) const + 139 at IORegistryEntry.cpp:1325
  - kernel`IORegistryEntry::attachToChild(IORegistryEntry*, IORegistryPlane const*) + 69 at IORegistryEntry.cpp:1695
  - kernel`IORegistryEntry::attachToParent(IORegistryEntry*, IORegistryPlane const*) + 537 at IORegistryEntry.cpp:1670
  - kernel`IOService::attach(IOService*) + 129 at IOService.cpp:574
  - UplinkOSXDevice`UplinkHub::setUpDevice(unsigned long long*) + 215 at UplinkHub.cpp:76
  - UplinkOSXDevice`UplinkHubClient::setUpDevice(unsigned long long*) + 108 at UplinkHubClient.cpp:161
  - UplinkOSXDevice`UplinkHubClient::dispatchSetUpDevice(UplinkHubClient*, void*, IOExternalMethodArguments*) + 76 at UplinkHubClient.cpp:204
  - UplinkOSXDevice`UplinkHubClient::externalMethod(unsigned int, IOExternalMethodArguments*, IOExternalMethodDispatch*, OSObject*, void*) + 141 at UplinkHubClient.cpp:237
  - kernel`::is_io_connect_method(io_connect_t, uint32_t, io_user_scalar_t *, mach_msg_type_number_t, char *, mach_msg_type_number_t, mach_vm_address_t, mach_vm_size_t, char *, mach_msg_type_number_t *, io_user_scalar_t *, mach_msg_type_number_t *, mach_vm_address_t, mach_vm_size_t *) + 487 at IOUserClient.cpp:3697
  - kernel`_Xio_connect_method + 384 at device_server.c:8255
  - kernel`ipc_kobject_server + 259 at ipc_kobject.c:340
  - kernel`ipc_kmsg_send + 168 at ipc_kmsg.c:1441
  - kernel`mach_msg_overwrite_trap + 197 at mach_msg.c:472
  - kernel`mach_call_munger64 + 410 at bsd_i386.c:560
  - kernel`hndl_mach_scall64 + 22
```

## Why?

I'm working on a tool to wirelessly stream audio from a mobile device to a desktop device.
Part of that tool involves writing a custom virtual audio driver in OS X.
I ran into a refcounting problem in my driver where one of the objects has an outstanding reference upon trying to shut down the device, preventing the driver from unloading.
Without a second Mac I can't set up remote kernel debugging to try and figure out what's going wrong.

But, I can add `panic()`s at convenient spots and examine the traces from the resulting kernel panics! :D

