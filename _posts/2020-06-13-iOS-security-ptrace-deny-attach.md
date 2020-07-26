---
title: "iOS security series: Anti-debugging techniques"
categories:
  - iOS Security
tags:
  - PTrace
  - antidebug
  - owasp
  - swift package
header:
  teaser: "assets/images/hackerimg.jpg"
---

The first part of the iOS security miniseries, where I'm describing my attempts to make apps more secure.

{% raw %}

<img src="../../assets/images/hackerimg.jpg" alt="">

{% endraw %}

When developing various banking apps, I came across some techniques to increase our app security.
No method is **100%** bulletproof, but our job is to make it harder for the attacker to penetrate.
One of the methods is to kill the app when the debugger is being attached.
We can achieve that by using a lower level system call, called `ptrace`.

## [Ptrace](https://www.man7.org/linux/man-pages/man2/ptrace.2.html)

According to the documentation `ptrace` (process trace) is a system call built inside kernel. It provides a mechanism by which one process can inspect, control, manipulate the execution of another process. It is used by debuggers and other code-analysis tools.

So how can we use that? 

Create `PtraceC.c` and `PtraceC.h` files.

Add to `PtraceC.h` this.
```c
void disable_gdb();
```
Add these lines to `PtraceC.c`
{% highlight c linenos %}
#include <stdio.h>
#import <dlfcn.h>
#import <sys/types.h>
#include "PtraceC.h"

typedef int (*ptrace_ptr_t)(int _request, pid_t _pid, caddr_t _addr, int _data);
#define PT_DENY_ATTACH 31

void disable_gdb() {
    ptrace_ptr_t ptrace_ptr = dlsym(RTLD_SELF, "ptrace");
    ptrace_ptr(31, 0, 0, 0); // PTRACE_DENY_ATTACH = 31
}
{% endhighlight %}

Now, let's walk through the code and understand what have we just done. 

- In line `6` we have defined a ptrace type. It takes in 4 parameters. We are only interested in the first parameter. `int _request`, which tells the function what to do. 
We use `PT_DENY_ATTACH 31` an apple-specific constant, which will prevent debuggers from debugging our binary.
- With `dlsym` function we will obtain the address to a `ptrace` call. Now calling `ptrace_ptr(PT_DENY_ATTACH, 0, 0, 0)` will send `SEGFAULT` to its tracing parent.
- With `RTLD_SELF` we specify that the search for symbols will begin from the object that invoked `dsym`.

The above implementation works, but thanks to these [guys](https://www.notsosecure.com/bypassing-jailbreak-detection-ios/) we know that it could be easily bypassed. So let's go deeper.

{% raw %}

<img src="../../assets/images/deeper.jpg" alt="">

{% endraw %}

## Assembly code
We will use inline assembly instructions to make direct system calls. In order to bypass this, the attacker must be somehow able to decrypt and override assembly instructions, that should be harder to crack.
```c
void disable_gdb() {
    __asm (
        "mov x0, #26\n" // ptrace
        "mov x1, #31\n" // PT_DENY_ATTACH
        "mov x2, #0\n"
        "mov x3, #0\n"
        "mov x16, #0\n"
        "svc #128\n"
      );
}
```
The above code is basically `ptrace_ptr(PT_DENY_ATTACH, 0, 0, 0)` written in **ARM64** assembly. Ptrace is listed as number `26` in system call. This code won't compile for simulators or **OS X** apps (our Macs are x86_64 Intel based) so add below assembly x86_64 implementation for other cases. 
```c
void disable_gdb() {
    #if defined (__arm64__)
    __asm(
        "mov x0, #26\n" // ptrace
        "mov x1, #31\n" // PT_DENY_ATTACH
        "mov x2, #0\n"
        "mov x3, #0\n"
        "mov x16, #0\n"
        "svc #128\n"
    );
    #elif defined(__x86_64__)
    __asm(
        "movq $0, %rcx\n"
        "movq $0, %rdx\n"
        "movq $0, %rsi\n"
        "movq $0x1f, %rdi\n"
        "movq $0x200001A, %rax\n"
        "syscall\n"
    );
    #endif
}
```
After adding little obfuscation to our `PtraceC.h` header file, it should look like this
```c
#define disable_gdb Dnnfg5Gh68k3Nmq2Evcr
void disable_gdb();
```

## Using C code in Swift Project
Now we have our C implementation. There are several ways to embed it into our Swift project. The easiest way is to include our `PtraceC.h` file to ObjC `YourProject-Bridging-Header.h`.

OR we can do it the fancy way! :wine_glass:

### Swift package :briefcase:
Open your terminal and type
```
mkdir Ptrace && cd Ptrace && swift package init --type library
```
The package should be initialized by now. Open your Xcode using
```
xed .
```
Now we will configure our `Package.swift` file.
```swift
import PackageDescription

let package = Package(
    name: "Ptrace",
    products: [
        .library(name: "Ptrace", targets: ["Ptrace"])
    ],
    dependencies: [],
    targets: [
        .target(
            name: "Ptrace",
            dependencies: ["PtraceC"],
            path: "Sources/PtraceSwift"
            ),
        .target(
            name: "PtraceC",
            path: "Sources/PtraceC"
            )
    ]
)
```
Our output library will be Ptrace. We will also make a tiny Swift wrapper.
Now create your project structure and files like this.
```
Ptrace
├── Package.swift
├── Sources
     ├──PtraceC
          ├── include
                ├── PtraceC.h
          └── PtraceC.c
     ├── PtraceSwift
          ├── PtraceSwift.swift
```
In PtraceSwift file, we will define a simple wrapper.
```swift
import PtraceC

public enum Ptrace {
    public static func denyAttach() {
        Dnnfg5Gh68k3Nmq2Evcr()
    }
}
```
We should be able to build the package now. The last step is to include this in our projects.
- Copy the package folder into your project directory
```
cp -R Ptrace toYourProjectFolder
```
- Drag the package folder into your project in Xcode
- In **Link Binary with Libraries** section, locate the package and select the gray library icon inside the package.

You should now be able to use it anywhere in your app and use it like this:
```swift
import Ptrace

Ptrace.dennyAttach()
```

Make sure to only use it in your app store releases.

## Source:

- [owasp](https://github.com/OWASP/owasp-mstg/blob/master/Document/0x06j-Testing-Resiliency-Against-Reverse-Engineering.md)
- [iphonedevwiki.net](http://iphonedevwiki.net/index.php/Crack_prevention#PT_DENY_ATTACH)
- Complete example [here](https://github.com/iaminh/Ptrace).
