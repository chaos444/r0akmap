# r0akmap

r0akmap is a (PoC) driver manual mapper based on r0ak, similar to Turla Driver Loader.
HVCI must be disabled for this to work.
Unfortunately due to paging and stuff the loader BSODs about 80% the time.

# r0ak![Downloads](https://img.shields.io/github/downloads/atom/atom/total.svg)

r0ak is a Windows command-line utility that enables you to easily read, write, and execute kernel-mode code (with some limitations) from the command prompt, without requiring anything else other than Administrator privileges.

## Quick Peek

```
r0ak v1.0.0 -- Ring 0 Army Knife
http://www.github.com/ionescu007/r0ak
Copyright (c) 2018 Alex Ionescu [@aionescu]
http://www.windows-internals.com

USAGE: r0ak.exe
       [--execute <Address | module.ext!function> <Argument>]
       [--write   <Address | module.ext!function> <Value>]
       [--read    <Address | module.ext!function> <Size>]
```

![Screenshot](r0ak-demo.png)

## Introduction

### Motivation

The Windows kernel is a rich environment in which hundreds of drivers execute on a typical system, and where thousands of variables containing global state are present. For advanced troubleshooting, IT experts will typically use tools such as the Windows Debugger (WinDbg), SysInternals Tools, or write their own. Unfortunately, usage of these tools is getting increasingly hard, and they are themselves limited by their own access to Windows APIs and exposed features.

Some of today's challenges include:

* Windows 8 and later support Secure Boot, which prevents kernel debugging (including local debugging) and loading of test-signed driver code. This restricts troubleshooting tools to those that have a signed kernel-mode driver.
* Even on systems without Secure Boot enabled, enabling local debugging or changing boot options which ease debugging capabilities will often trigger BitLocker's recovery mode.
* Windows 10 Anniversary Update and later include much stricter driver signature requirements, which now enforce Microsoft EV Attestation Signing. This restricts the freedom of software developers as generic "read-write-everything" drivers are frowned upon.
* Windows 10 Spring Update now includes customer-facing options for enabling HyperVisor Code Integrity (HVCI) which further restricts allowable drivers and blacklists multiple 3rd party drivers that had "read-write-everything" capabilities due to poorly written interfaces and security risks.
* Technologies like Supervisor Mode Execution Prevention (SMEP), Kernel Control Flow Guard (KCFG) and HVCI with Second Level Address Translation (SLAT) are making traditional Ring 0 execution 'tricks' obsoleted, so a new approach is needed.

In such an environment, it was clear that a simple tool which can be used as an emergency band-aid/hotfix and to quickly troubleshoot kernel/system-level issues which may be apparent by analyzing kernel state might be valuable for the community.

### How it Works

#### Basic Architecture

![Diagram](r0ak-archdiag.png)

r0ak works by redirecting the execution flow of the window manager's trusted font validation checks when attempting to load a new font, by replacing the trusted font table's comparator routine with an alternate function which schedules an executive work item (`WORK_QUEUE_ITEM`) stored in the input node. Then, the trusted font table's right child (which serves as the root node) is overwritten with a named pipe's write buffer (`NP_DATA_ENTRY`) in which a custom work item is stored. This item's underlying worker function and its parameter are what will eventually be executed by a dedicated `ExpWorkerThread` at `PASSIVE_LEVEL` once a font load is attempted and the comparator routine executes, receiving the name pipe-backed parent node as its input. A real-time Event Tracing for Windows (ETW) trace event is used to receive an asynchronous notification that the work item has finished executing, which makes it safe to tear down the structures, free the kernel-mode buffers, and restore normal operation.

#### Supported Commands

When using the `--execute` option, this function and parameter are supplied by the user.

When using `--write`, a custom gadget is used to modify arbitrary 32-bit values anywhere in kernel memory.

When using `--read`, the write gadget is used to modify the system's HSTI buffer pointer and size (__**N.B.: This is destructive behavior in terms of any other applications that will request the HSTI data. As this is optional Windows behavior, and this tool is meant for emergency debugging/experimentation, this loss of data was considered acceptable**__). Then, the HSTI Query API is used to copy back into the tool's user-mode address space, and a hex dump is shown.

Because only built-in, Microsoft-signed, Windows functionality is used, and all called functions are part of the KCFG bitmap, there is no violation of any security checks, and no debugging flags are required, or usage of 3rd party poorly-written drivers.

### FAQ

#### Is this a bug/vulnerability in Windows?

No. Since this tool -- and the underlying technique -- require a SYSTEM-level privileged token, which can only be obtained by a user running under the Administrator account, no security boundaries are being bypassed in order to achieve the effect. The behavior and utility of the tool is only possible due to the elevated/privileged security context of the Administrator account on Windows, and is understood to be a by-design behavior.

#### Was Microsoft notified about this behavior?

Of course! It's important to always file security issues with Microsoft even when no violation of privileged boundaries seems to have occurred -- their teams of researchers and developers might find novel vectors and ways to reach certain code paths which an external researcher may not have thought of.

As such, in November 2014, a security case was filed with the Microsoft Security Research Centre (MSRC) which responded:
"*[…] doesn't fall into the scope of a security issue we would address via our traditional Security Bulletin vehicle. It […] pre-supposes admin privileges -- a place where architecturally, we do not currently define a defensible security boundary. As such, we won't be pursuing this to fix.*"

Furthermore, in April 2015 at the Infiltrate conference, a talk titled *Insection : AWEsomely Exploiting Shared Memory Objects* was presented detailing this issue, including to Microsoft developers in attendance, which agreed this was currently out of scope of Windows's architectural security boundaries. This is because there are literally dozens -- if not more -- of other ways an Administrator can read/write/execute Ring 0 memory. This tool merely allows an easy commodification of one such vector, for purposes of debugging and troubleshooting system issues.

#### Can't this be packaged up as part of end-to-end attack/exploit kit?

Packaging this code up as a library would require carefully removing all interactive command-line parsing and standard output, at which point, without major rewrites, the 'kit' would:

* Require the target machine to be running Windows 10 Anniversary Update x64 or later
* Have already elevated privileges to SYSTEM
* Require an active Internet connection with a proxy/firewall allowing access to Microsoft's Symbol Server
* Require the Windows SDK/WDK installed on the target machine
* Require a sensible _NT_SYMBOL_PATH environment variable to have been configured on the target machine, and for about 15MB of symbol data to be downloaded and cached as PDB files somewhere on the disk

Attackers interested in using this particular approach -- versus very many others more cross-compatible, no-SYSTEM-right-requiring techniques -- likely already adapted their own code based on the Proof-of-Concept from April 2015 -- more than 3 years ago.

## Usage

### Requirements

Due to the usage of the Windows Symbol Engine, you must have either the Windows Software Development Kit (SDK) or Windows Driver Kit (WDK) installed with the Debugging Tools for Windows. The tool will lookup your installation path automatically, and leverage the `DbgHelp.dll` and `SymSrv.dll` that are present in that directory. As these files are not re-distributable, they cannot be included with the release of the tool.

Alternatively, if you obtain these libraries on your own, you can modify the source-code to use them.

Usage of symbols requires an Internet connection, unless you have pre-cached them locally. Additionally, you should setup the `_NT_SYMBOL_PATH` variable pointing to an appropriate symbol server and cached location.

It is assumed that an IT Expert or other troubleshooter which apparently has a need to read/write/execute kernel memory (and has knowledge of the appropriate kernel variables to access) is already more than intimately familiar with the above setup requirements. Please do not file issues asking what the SDK is or how to set an environment variable.

### Use Cases

* Some driver leaked kernel pool? Why not call `ntoskrnl.exe!ExFreePool` and pass in the kernel address that's leaking? What about an object reference? Go call `ntoskrnl.exe!ObfDereferenceObject` and have that cleaned up.

* Want to dump the kernel DbgPrint log? Why not dump the internal circular buffer at `ntoskrnl.exe!KdPrintCircularBuffer`

* Wondering how big the kernel stacks are on your machine? Try looking at `ntoskrnl.exe!KeKernelStackSize`

* Want to dump the system call table to look for hooks? Go print out `ntoskrnl.exe!KiServiceTable`

These are only a few examples -- all Ring 0 addresses are accepted, either by `module!symbol` syntax or directly passing the kernel pointer if known. The Windows Symbol Engine is used to look these up.

### Limitations

The tool requires certain kernel variables and functions that are only known to exist in modern versions of Windows 10, and was only meant to work on 64-bit systems. These limitations are due to the fact that on older systems (or x86 systems), these stricter security requirements don't exist, and as such, more traditional approaches can be used instead. This is a personal tool which I am making available, and I had no need for these older systems, where I could use a simple driver instead. That being said, this repository accepts pull requests, if anyone is interested in porting it.

Secondly, due to the use cases and my own needs, the following restrictions apply:
* Reads -- Limited to 4 GB of data at a time
* Writes -- Limited to 32-bits of data at a time
* Executes -- Limited to functions which only take 1 scalar parameter

Obviously, these limitations could be fixed by programmatically choosing a different approach, but they fit the needs of a command line tool and my use cases. Again, pull requests are accepted if others wish to contribute their own additions.

Note that all execution (including execution of the `--read` and `--write` commands) occurs in the context of a System Worker Thread at `PASSIVE_LEVEL`. Therefore, user-mode addresses should not be passed in as parameters/arguments.

## Contributing

Pull requests are welcome. For major changes, please open an issue first to discuss what you would like to change.

## License
```
Copyright 2018 Alex Ionescu. All rights reserved. 

Redistribution and use in source and binary forms, with or without modification, are permitted provided
that the following conditions are met: 
1. Redistributions of source code must retain the above copyright notice, this list of conditions and
   the following disclaimer. 
2. Redistributions in binary form must reproduce the above copyright notice, this list of conditions
   and the following disclaimer in the documentation and/or other materials provided with the 
   distribution. 

THIS SOFTWARE IS PROVIDED BY ALEX IONESCU ``AS IS'' AND ANY EXPRESS OR IMPLIED
WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE IMPLIED WARRANTIES OF MERCHANTABILITY AND
FITNESS FOR A PARTICULAR PURPOSE ARE DISCLAIMED. IN NO EVENT SHALL ALEX IONESCU
OR CONTRIBUTORS BE LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL, EXEMPLARY, OR
CONSEQUENTIAL DAMAGES (INCLUDING, BUT NOT LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS
OR SERVICES; LOSS OF USE, DATA, OR PROFITS; OR BUSINESS INTERRUPTION) HOWEVER CAUSED
AND ON ANY THEORY OF LIABILITY, WHETHER IN CONTRACT, STRICT LIABILITY, OR TORT (INCLUDING
NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF THE USE OF THIS SOFTWARE, EVEN IF
ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.

The views and conclusions contained in the software and documentation are those of the authors and
should not be interpreted as representing official policies, either expressed or implied, of Alex Ionescu.
