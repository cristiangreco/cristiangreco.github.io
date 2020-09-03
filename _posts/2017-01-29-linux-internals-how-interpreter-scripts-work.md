---
layout: post
title: "Linux internals: how interpreter scripts work"
date: 2017-01-29 20:00:00
tags: linux scripting, linux, shell, interpreter, bash, path, setuid
comments: true
excerpt: |-
  How do scripts work on Linux? How does the kernel find the correct interpreter
  for a script and run it with arguments? In this post we'll deep-dive into kernel
  code that handles scripts execution and will show some of the Linux
  peculiarities with practical examples.
---

Shell scripts are a common topic in Linux interview questions. Beside mastering
the [art of Bash scripting](http://tldp.org/LDP/abs/html/){:target="_blank"}, it
is enlightening to understand how the kernel treats a script - any kind of
interpreter script - differently than, say, an ordinary ELF executable.

### Definitions, first!

A script is an executable file that begins with a line (starting with the `#!`
characters) specifying a path to a script interpreter.

`#! /path/to/interpreter [ args ]`

On most Unix implementations, the space after the `#!` is optional.

### How the kernel executes a script

What happens when you execute a Python script like this one?

```python
#!/usr/bin/python

print "Hello World"
```

If you `strace` the script to intercept syscalls, this is what you should see:

```
$ strace -e trace=open,execve,write ./hello.py
execve("./hello.py", ["./hello.py"], [/* 59 vars */]) = 0
open("/etc/ld.so.cache", O_RDONLY|O_CLOEXEC) = 3
open("/lib/x86_64-linux-gnu/libpthread.so.0", O_RDONLY|O_CLOEXEC) = 3
open("/lib/x86_64-linux-gnu/libdl.so.2", O_RDONLY|O_CLOEXEC) = 3
open("/lib/x86_64-linux-gnu/libutil.so.1", O_RDONLY|O_CLOEXEC) = 3
open("/lib/x86_64-linux-gnu/libz.so.1", O_RDONLY|O_CLOEXEC) = 3
open("/lib/x86_64-linux-gnu/libm.so.6", O_RDONLY|O_CLOEXEC) = 3
open("/lib/x86_64-linux-gnu/libc.so.6", O_RDONLY|O_CLOEXEC) = 3
open("/usr/lib/python2.7/site.x86_64-linux-gnu.so", O_RDONLY) = -1 ENOENT (No such file or directory)
...
open("./hello.py", O_RDONLY)             = 3
write(1, "Hello World\n", 12Hello World
)           = 12
+++ exited with 0 +++
```

There is exactly one `execve()` call, with the script filename as first
argument: no Python interpreter is being executed here!

Well, what happens [under the
hood](http://lxr.free-electrons.com/source/fs/binfmt_script.c){:target="_blank"}
is the kernel recognizing such executable file as an interpreter script, given
it starts with the sha-bang magic numbers (`0x23 0x21`). The whole first line of
the file is then split into an interpreter path and (optional) args. The
interpreter is located on the filesystem and invoked with the script path as
first argument.

In the example above, any syscall after the `execve()` is actually issued by the
Python interpreter binary, although you can't directly see it being executed.

### Interpreter and script's arguments

The first line of a script file specifies the interpreter path and optional
arguments for the interpreter itself. Any other command line argument is
appended after the script path. Let's explain this with an example:

```
$ cat script.sh
#!/home/cristian/myecho interp_arg1 interp_arg2
```

The `myecho` interpreter is a little C program that outputs `argv`:

```c
#include <stdio.h>

int main(int argc, char* argv[]) {
  for (int i = 0; i < argc; i++) {
    printf("%d -> %s\n", i, argv[i]);
  }
}
```

When executing `script.sh` with additional command line arguments, output is:

```
$ ./script.sh cli_arg1 cli_arg2
0 -> /home/cristian/myecho
1 -> interp_arg1 interp_arg2
2 -> ./script.sh
3 -> cli_arg1
4 -> cli_arg2
```

Here you see that inside `execve()` the arguments are re-arranged in this
manner:

```
interpreter-path interpreter-args script-path script-args
```

On Linux, the interpreter args are parsed as a single line (this explains why
`interp_arg1 interp_arg2` are printed as a single `argv` entry), and this may be
the source of portability issues. You should also be aware that, on Linux, the
total length of the first line of a script (i.e. sha-bang + interpreter-path +
interpreter-args) cannot be longer than [128
characters](http://lxr.free-electrons.com/source/include/uapi/linux/binfmts.h){:target="_blank"}
(including newline).

### The problem with PATH

The interpreter path is usually an absolute path (although a relative path can
still be used). In fact, the `$PATH` mechanism of your shell is meaningless in
kernel space (remember everything we're discussing here happens inside an
`execve()` syscall). The interpreter path must be the exact location of an
executable file.

For the sake of portability, the `env(1)` utility is often used as a workaround.
Assuming that `env` is commonly installed under the standard `/usr/bin/env`
path, this small executable is then used as a trampoline to start the real
interpreter, which then needs to be located somewhere in the user's PATH.

```python
#!/usr/bin/env python

print "Hello World"
```

In other words, `#!/usr/bin/env python` executes `env` as an interpreter,
passing `python` as an interpreter-arg. `env` itself is
[usually](http://git.savannah.gnu.org/gitweb/?p=coreutils.git;a=blob;f=src/env.c){:target="_blank"}
[implemented](https://git.busybox.net/busybox/tree/coreutils/env.c){:target="_blank"}
with an `execvp(3)` call - a C library call - that searches for the given
executable in PATH.

### Nesting the interpreter

Some UNIX implementations permit the interpreter of a script to itself be a
script. On Linux, this kind of "executable search" is recursive up to an
hardcoded limit of [four
recursions](http://lxr.free-electrons.com/source/fs/exec.c#L1560){:target="_blank"}.


```
$ cat nested.sh
#!./nested.sh

$ ./nested.sh
bash: ./nested.sh: /nested.sh: bad interpreter: Too many levels of symbolic links
```

### The setuid bit is ignored

On Linux, and in other Unix implementations, the `setuid` bit on scripts in
ignored for security reasons.

We can prepare a small test program:

```c
#include <unistd.h>
#include <stdio.h>

int main() {
  printf("real=%d effective=%d\n", getuid(), geteuid());
}
```

The `setuid` permission on the binary executable works as expected:

```
$ ./test-setuid
real=1000 effective=1000

$ sudo chown root test-setuid && sudo chmod +s test-setuid

$ ./test-setuid
real=1000 effective=0
```

If we do the same job on a script, the `setuid` bit is ignored instead:

```
$ cat test-setuid.sh
#!/bin/bash

id -u

$ ./test-setuid.sh
1000

$ sudo chown root test-setuid.sh && sudo chmod +s test-setuid.sh

$ ./test-setuid.sh
1000
```

### Further reading

In the description above, I've mentioned a few times that some details are very
Linux-specific. This is because, unfortunately, the `#!` mechanism is not
officially part of any Unix specification.
[Here](http://www.in-ulm.de/~mascheck/various/shebang/){:target="_blank"} are
described more in-depth details about the differences between existing Unix
flavours and how they evolved over time.
