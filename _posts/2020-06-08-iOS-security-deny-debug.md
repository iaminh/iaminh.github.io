---
title: "iOS security series: Anti-debugging in apps"
categories:
  - iOS Security
tags:
  - PTrace
  - deny debugger
  - swift package
---

First article of the iOS security miniseries, where I will describe how I made iOS apps more secure. Hope it can also help you.

When developing various banking apps, we always have to have in mind our beloved users security.
No method is 100% bulletproof, but our job is to make it harder for the attacker to penetrate.
One of the method is to kill the app, when debugger is attached.
We can achieve that by using lower level C function, called `ptrace`.

## [Ptrace](https://www.man7.org/linux/man-pages/man2/ptrace.2.html)
Accoring to the documentation `ptrace` (process trace) is a system call built inside kernel. It provides a mechanism by which one process can inspect, control, manipulate the execution of another process. It is used by debuggers and other code-analysis tools.

So how can we use that? Create a `PtraceC.c` and `PtraceC.h` and add to `.c` file these few lines of code

{% highlight c linenos %}
#include <stdio.h>
#import <dlfcn.h>
#import <sys/types.h>
#include "PtraceC.h"

typedef int (*ptrace_ptr_t)(int _request, pid_t _pid, caddr_t _addr, int _data);
#define PT_DENY_ATTACH 31

void disable_gdb() {
    void* handle = dlopen(0, RTLD_GLOBAL | RTLD_NOW);
    ptrace_ptr_t ptrace_ptr = dlsym(handle, "ptrace");
    ptrace_ptr(PT_DENY_ATTACH, 0, 0, 0);
    dlclose(handle);
}
{% endhighlight %}

Now, lets walkthrough the code and understand what have we just done. 

- On line `x` we have defined a ptrace type. It takes in 4 parameters. We are only interested in only the first parameter. `int _request`, which will tell the function what to do - `PT_DENY_ATTACH 31`, Apple-specific constant, that will prevent debuggers from debugging our binary.
- On line `x` we open a dynamic library for the main program with available symbols. This returns us a handler object which we will use in the next function `dlsym`.
- With `dlsym` function we will obtain the address to a `ptrace` call. Now calling `ptrace_ptr(PT_DENY_ATTACH, 0, 0, 0)` will send `SEGFAULT` to its tracing parent.
- After that we inform the system that we no longer need the dynamic library handle with `dlclose`.

The above implementation works, but thanks to these [guys](http://iphonedevwiki.net/index.php/Crack_prevention#PT_DENY_ATTACH) we know that it could could be easily bypassed. So let's go deeper.

{% raw %}

<img src="../../assets/images/deeper.jpg" alt="">

{% endraw %}

### Assembly code
Replace the above code with:
```c
void disable_gdb() {
    __asm (
           "mov r0, #31\n"
           "mov r1, #0\n"
           "mov r2, #0\n"
           "mov r3, #0\n"
           "mov ip, #26\n"
           "svc #0x80\n"
           );
}
```

## Using C code in Swift Project
Now we have our C implementation. There are several ways to embed it to our Swift project. The easiest way is to include our `PtraceC.h` file to `Bridging-Header.h`. But we can do it a fancy way!

### Swift package
Let's create a swift package.

Source:

- https://sushi2k.gitbooks.io/the-owasp-mobile-security-testing-guide/content/0x06j-Testing-Resiliency-Against-Reverse-Engineering.html
