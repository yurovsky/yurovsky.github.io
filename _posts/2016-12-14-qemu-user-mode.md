---
layout: post
title: Using qemu user mode to run cross-compiled binaries
---

The qemu CPU emulator is typically used to emulate an entire system, for example
we can use `qemu-system-arm` to start a simulated ARM-based machine, boot Linux, and then run whatever software is appropriate. There is also a user mode which lets us run a cross-compiled executable right on our host machine and that can be used for various isolated testing tasks (for instance, running unit tests in the target system architecture).

A useful program does need some libraries however and it may need an environment.  qemu user mode takes a number of arguments that can be used to supply these. The `-L` option lets us specify the path for the ELF interpreter prefix and we can utilize this to point the emulator to a reasonable set of libraries (for example a target file system sitting on my x86 laptop).  Additional environment variables can be set with `-E`.

## Running a cross-compiled program

I have built a BSP using [buildroot](https://buildroot.org/) and now have a `target/` directory that contains nearly everything that will wind up on my root file system image (but note [a few caveats](https://buildroot.org/downloads/manual/manual.html#faq-why-not-use-target-as-chroot)) for this target. I can take advantage of the fact that `target/` has everything that is needed to run an executable (that is, there's a `/lib` directory there with libraries in the target architecture) so now I can use qemu to run an executable.

For instance,

    qemu-arm -L target/ /path/to/some/executable

Or to run the cross-compiled ARM `/bin/ls` right from the target directory,

    qemu-arm -L target/ target/bin/ls

should work provided the ELF interpreter prefix matches up. In my case `ls` is a symlink to `busybox` and running `file` on that produces:

    file target/bin/busybox 
    target/bin/busybox: ELF 32-bit LSB executable, ARM, EABI5 version 1 (SYSV), dynamically linked, interpreter /lib/ld-linux.so.3, for GNU/Linux 4.7.0, not stripped

and target/ has `lib/ld-linux.so.3` present so passing `-L target` gets me a running executable.

In addition, `$?` will report the cross-compiled program's return code correctly so with the help of the target ELF interpreter and libraries I can now run unit tests after building my BSP, those unit tests can run in their native architecture, and I can capture output from them and utilize their return codes (for example via `make check` in autotools) like I would expect with native unit tests. Furthermore, the unit test executables themselves never need to leave my build system (that is, they don't need to wind up on the target root file system or even an emulated device) and I do not need a special build just for running those tests.

## Unit Testing

There's arguably value in running unit tests on their target CPU (even if that CPU is emulated), for example:

* Memory alignment will match reality (x86 supports unaligned memory access, ARMv5 does not and we can write C code that will not work on ARMv5)
* Math behaves the same (consider a 64-bit x86 machine performing 64-bit arithmetic vs. a 32-bit ARM machine performing the same arithmetic, we can write C programs that will give different results on different CPUs)
* We can run tests in their expected target endianness and that can matter even though carefully written code should behave correctly.

Unit tests run with qemu user mode would ideally be real testable units since we're not simulating the entire machine (that is, they should not interact with hardware or other software since they are running on their own) but that is in line with unit testing practices anyhow.  Functional tests that need additional software components or hardware can be tested on an emulated machine with qemu system mode.

## Interactive debugging with GDB

qemu user mode can act as a GDB server, enabling us to debug our cross-compiled program.  Consider the usual trivial program:

    #include <stdio.h>

    int main(void)
    {
        printf("Hello, world!\n");
        return 0;
    }

We can compile this with debug symbols via our toolchain, for example:

    arm-buildroot-linux-gnueabi-gcc -g -c hello.c -o hello.o
    arm-buildroot-linux-gnueabi-gcc hello.o -o hello

Start qemu with a GDB server on an arbitrary port, for example 12345:

    qemu-arm -L target/ -g 12345 ./hello

and it will wait for GDB to attach.  We can then run our toolchain GDB (using the debug symbols in `hello`):

    arm-buildroot-linux-gnueabi-gdb ./hello

and then, in GDB's console, connect to the GDB server:

    target remote localhost:12345

we should also tell GDB to use our target root file system as the sysroot (since that's where all the libraries are), this enables GDB to find libc and anything else we may depend on:

    set sysroot ./target

We can now set breakpoints and debug like we would any other program, for example:

    b main
    c

We're now able to run and debug our cross-compiled program without needing to boot a real or even emulated machine, provided the program can run on its own without needing access to resources that qemu user mode won't provide (again in those cases, use `qemu-system-arm` to boot an emulated machine running Linux).
