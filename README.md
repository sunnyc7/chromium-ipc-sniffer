# Chromium IPC Sniffer
This utility helps you explore what Chrome processes are saying to each other under the hood in real-time, using Wireshark.

It captures data sent over the [Named Pipe](https://docs.microsoft.com/en-us/windows/win32/ipc/named-pipes) IPC primitive and sends it over to dissection.

<img src="https://raw.githubusercontent.com/tomer8007/chromium-ipc-sniffer/master/screenshot_3.png" >

## Supported protocol and formats
* [Mojo Core](https://chromium.googlesource.com/chromium/src/+/master/mojo/core/README.md) (Ports, Nodes, Invitations, Handles, etc.)
* [Mojo binded user messages](https://chromium.googlesource.com/chromium/src/+/master/mojo/public/cpp/bindings/README.md) (actual `.mojom` IDL method calls)
* [Legacy IPC](https://www.chromium.org/developers/design-documents/inter-process-communication)
* [Mojo data pipe](https://chromium.googlesource.com/chromium/src/+/master/mojo/public/c/system/README.md#Data-Pipes) control messages (read/wrote X bytes)
* Audio sync messages (`\pipe\chrome.sync.xxxxx`)

However, this project won't see anything that doesn't go over pipes, which is mostly shared memory IPC:
* Mojo data pipe contents (real time networking buffers, audio, etc.)
* [Sandbox IPC](https://chromium.googlesource.com/chromium/src/+/master/docs/design/sandbox.md#the-target-process)
* Possibly more things

## Usage
You can download pre-compiled binaries from the [Releases](https://github.com/tomer8007/chromium-ipc-sniffer/releases) page, and run:
```
C:\>chromeipc.exe

Chrome IPC Sniffer v0.5.0.0

Type -h to get usage help and extended options

[+] Starting up
[+] Determining your chromium version
[+] You are using chromium 83.0.4103.116
[+] Checking mojom interfaces information
[+] Checking legacy IPC interfaces information
[+] Extracting scrambled message IDs from chrome.dll...
[+] Copying LUA dissectors to Wirehsark plugins directory
[+] Enumerating existing chrome pipes
[+] Starting sniffing of chrome named pipe to \\.\pipe\chromeipc.
[+] Opening Wirehark
[+] Capturing 40 packets/second......
```

Wireshark should open automatically.

## Cheat Sheet
### Filtering
It's worth noting that you can filter the results in Wireshark to show only packets of interest. 
Examples:
* To show only packets going to/from a particular process, use `npfs.pid == 1234`
* To show only packets with a particular method name, use `mojouser.method contains "SomeMethod"`

### Enabling deep mojo arguments dissection
By default, the LUA dissectors will only show `Nested Struct/Array` trees and won't try to go through all the fields.
You can enable deep inspection, but it's slow for a large number of packets and not complete.

Go to Edit -> Prefrences -> Protocols -> MOJOUSER -> Enable structs deep dissection

## Limitations
* Supports Chrome 80+ on 64-bit Windows only
* Tested only on official, branded Chrome builds. Could theoretically work on other builds too, as well as other chromium-based browsers (Edge)
* Names of methods as shown in Wireshark is based on the chromium sources, and some mojom interfaces use unscrambled ordinals, which won't be resolved
* Parsing is not 100% complete, e.g unions/enums/maps are not fully supported

## FAQ

### What is `tdevmonc.sys`?
`tdevmonc.sys` (or [Tibbo](https://tibbo.com/) Device Monitor) is a third-party kernel-mode driver that is used to capture the Named Pipe traffic.
The reason to include it is to avoid the need to enable test signing or to tampter with chrome processes.


The driver works by `IoAttachDeviceToDeviceStack`ing on top of the `\Device\NamedPipe` device and acting as a filter driver. Then the data that is written to pipes is exposed to user mode using various IOCTLs.

You can find sources for this driver [here](https://tibbo.com/downloads/archive/tdevmon/tdevmon-3.3.5/), as well as binaries and PDB [here](https://tibbo.com/downloads/archive/tdevmon/tdevmon-3.3.2/).

Note that this driver is used by [IO ninja](https://ioninja.com/), which is not entirely freeware.
Also note this driver does not practicly support unloading once it attaches to at least one device (you need to reboot).

