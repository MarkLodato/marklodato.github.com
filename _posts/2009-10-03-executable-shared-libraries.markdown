---
layout: post
title: Executable Shared Libraries
---

I figured out a way to create a shared library that is also executable.  One
possible use for this would be to make [Cython](http://cython.org) modules
that can be run by themselves or imported from Python.  I found some mailing
list discussions from 2003 describing a method, but it does not work on x86
Linux.  The method below works, at least, on x86-64 Linux with gcc 4.3.3 on
Ubuntu 9.04.

Unfortunately, I could not get gcc to do this for me - I have to run the
linker myself.  I tried giving `-Wl,-shared`, but gcc put the flag too late in
the command line.  If anyone figures out a way to do this, let me know!  

Here's some example code:

{% highlight bash %}
$ cat <<EOF >library.c
#include <stdio.h>

void foo(int x) {
    printf("foo(%d)\n", x);
}
EOF

$ cat <<EOF >main.c
#include <stdio.h>
#include <stdlib.h>

void foo(int x);

int main(int argc, char **argv) {
    if (argc != 2) {
        fprintf(stderr, "USAGE: %s n\n", argv[0]);
        exit(1);
    }
    foo(atoi(argv[1]));
    return 0;
}
EOF
{% endhighlight %}

Compile your objects with `-fPIC` and `-fPIE`:

    {% highlight bash %}
$ gcc -fPIC -fPIE -c library.c
$ gcc -fPIC -fPIE -c main.c
{% endhighlight %}

Run `gcc '-###'` to find out what command it *would* have run.

{% highlight bash %}
$ gcc '-###' -fPIC -fPIE -pie -rdynamic -o library.so library.o main.o
{% endhighlight %}

Look for the `collect2` line - this is the linker.  Run that exact command,
but insert `-shared` before the `-pie`.  On my system, I ran this:

{% highlight bash %}
$ /usr/lib/gcc/x86_64-linux-gnu/4.3.3/collect2 \
    --eh-frame-hdr \
    -m elf_x86_64 \
    -shared \
    -pie \
    --export-dynamic \
    -o library.so \
    -z relro \
    /usr/lib/Scrt1.o \
    /usr/lib/crti.o \
    /usr/lib/gcc/x86_64-linux-gnu/4.3.3/crtbeginS.o \
    -L/usr/lib/gcc/x86_64-linux-gnu/4.3.3 \
    -L/usr/lib \
    -L/lib \
    library.o \
    main.o \
    -lgcc \
    --as-needed \
    -lgcc_s \
    --no-as-needed \
    -lc \
    -lgcc \
    --as-needed \
    -lgcc_s \
    --no-as-needed \
    /usr/lib/gcc/x86_64-linux-gnu/4.3.3/crtendS.o \
    /usr/lib/crtn.o
{% endhighlight %}

Now you can both execute and load your shared library.

{% highlight bash %}
$ ./library.so 12
foo(12)

$ python -c \
    'import ctypes; x = ctypes.cdll.LoadLibrary("./library.so"); x.foo(13)'
foo(13)
{% endhighlight %}
