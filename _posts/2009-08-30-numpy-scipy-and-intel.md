---
layout: post
title: NumPy, SciPy, and the Intel Compiler Suite
---

I spent a *lot* of time trying to get [NumPy](https://numpy.scipy.org) and
[SciPy](https://www.scipy.org) to work with the [Intel Compiler
Suite](https://software.intel.com/en-us/intel-compilers/), but it finally
works.  Here is how I did it on Ubuntu 9.04/AMD64 on an Intel Core 2 Duo using
the versions specified.  It appears that every combination of versions requires
a different process, so hopefully this works for you!

### Intel Compiler Suite 11.1.046

Download both the Intel C++ and Fortran compilers for Linux, which are free
for non-commercial use.  I used Intel 64 version 11.1.046.  The install is
very easy; I installed the C++ compiler and then the Fortran one.

Once they are installed, source the iccvars.sh script.

```bash
source /opt/intel/Compiler/11.1/046/bin/iccvars.sh intel64
```

Also, you may want to do the following so that you do not need to have the
Intel library directories in your `$LD_LIBRARY_PATH`.

```bash
cat > /etc/ld.so.conf.d/intel.conf <<EOF
/opt/intel/Compiler/11.1/046/lib/intel64
/opt/intel/Compiler/11.1/046/ipp/em64t/sharedlib
/opt/intel/Compiler/11.1/046/mkl/lib/em64t
/opt/intel/Compiler/11.1/046/tbb/em64t/cc4.1.0_libc2.4_kernel2.6.16.21/lib
EOF
sudo /sbin/ldconfig
```

#### Note About -xHost

In recent Intel compilers, you can give the `-xHost` option, which will
optimize based on the Intel processor of the current host.  This works great
if you are compiling with an Intel processor and you will run on the same
machine.  This was my case, so this is the flag I used.  If this is not the
case (AMD processor, or compiling to run on other chips), then you will want
to set this optimization flag to something more appropriate.

### AMD / UMFPACK, part of SuiteSparse 3.4.0

The AMD and UMFPACK libraries (part of
[SuiteSparse](https://www.cise.ufl.edu/research/sparse/SuiteSparse/))
are needed for SciPy, so we'll build them first.  They are static libraries,
so I chose to build them in /opt.

```bash
cd /opt
tar zxvf SuiteSparse-3.4.0.tar.gz
cd SuiteSparse
```

First we need to patch the Makefile to build with the Intel compiler.  If you
want to compile the whole suite, you also need to set up BLAS and LAPACK to
use MKL.  But we don't need the whole suite for SciPy.

```diff
patch -p0 <<EOF
diff -ur SuiteSparse.orig/UFconfig/UFconfig.mk SuiteSparse/UFconfig/UFconfig.mk
--- SuiteSparse.orig/UFconfig/UFconfig.mk	2009-05-20 14:06:04.000000000 -0400
+++ SuiteSparse/UFconfig/UFconfig.mk	2009-08-07 02:03:19.000000000 -0400
@@ -33,11 +33,11 @@
 # C compiler and compiler flags:  These will normally not give you optimal
 # performance.  You should select the optimization parameters that are best
 # for your system.  On Linux, use "CFLAGS = -O3 -fexceptions" for example.
-CC = cc
+CC = icc
 # CFLAGS = -O   (for example; see below for details)

 # C++ compiler (also uses CFLAGS)
-CPLUSPLUS = g++
+CPLUSPLUS = icpc

 # ranlib, and ar, for generating libraries
 RANLIB = ranlib
@@ -48,8 +48,8 @@
 MV = mv -f

 # Fortran compiler (not normally required)
-F77 = f77
-F77FLAGS = -O
+F77 = ifort
+F77FLAGS = -O3 -xHost
 F77LIB =

 # C and Fortran libraries
@@ -220,7 +224,7 @@

 # Using default compilers:
 # CC = gcc
-CFLAGS = -O3 -fexceptions
+CFLAGS = -O3 -xHost -fPIC -openmp -vec_report=0

 # alternatives:
 # CFLAGS = -g -fexceptions \
EOF
```

Compile both libraries.  No install needed after this.

```bash
make -C AMD
make -C UMFPACK
```


### Numpy 1.3.0

Now on to NumPy.

```bash
tar zxvf numpy-1.3.0.tar.gz
cd numpy-1.3.0
```

Next, set up the site.cfg file for Intel MKL, AMD, and UMFPACK.  (Only MKL is
needed for NumPy, but if we set this up now, it will work for SciPy later.)

**Warning:** I used `mkl_mc` because I have a Core architecture.  Read the
[MKL User's Guide](file:///opt/intel/Compiler/11.1/046/Documentation/en_US/mkl/userguide.pdf)
to determine which kernel library to use for your architecture.  It *should*
be automatic (from `mkl_core`), but for some reason it kept failing with
`undefined symbol: mkl_dft_commit_descriptor_s_c2c_md_omp`.  Adding `mkl_mc`
fixed it.

```bash
cat > site.cfg <<EOF
[DEFAULT]
include_dirs = /opt/SuiteSparse/UFconfig

[amd]
amd_libs = amd
library_dirs = /opt/SuiteSparse/AMD/Lib
include_dirs = /opt/SuiteSparse/AMD/Include

[umfpack]
umfpack_libs = umfpack
library_dirs = /opt/SuiteSparse/UMFPACK/Lib
include_dirs = /opt/SuiteSparse/UMFPACK/Include

[mkl]
include_dirs = /opt/intel/Compiler/11.1/046/mkl/include
library_dirs = /opt/intel/Compiler/11.1/046/mkl/lib/em64t
lapack_libs = mkl_lapack
mkl_libs = mkl_intel_lp64, mkl_intel_thread, mkl_core, mkl_mc
EOF
```

Now, turn on optimization, `-fPIC`, and OpenMP support for `icc`.

```diff
patch -p0 <<EOF
--- numpy/distutils/intelccompiler.py.old       2009-03-29 07:24:21.000000000 -0400
+++ numpy/distutils/intelccompiler.py   2009-08-05 23:58:30.000000000 -0400
@@ -8,7 +8,7 @@
     """

     compiler_type = 'intel'
-    cc_exe = 'icc'
+    cc_exe = 'icc -xHost -O3 -fPIC -openmp'

     def __init__ (self, verbose=0, dry_run=0, force=0):
         UnixCCompiler.__init__ (self, verbose,dry_run, force)
EOF
```

Set the appropriate flags for `ifort`, skipping Numpy's broken autodetection.

```diff
patch -p0 <<EOF
--- numpy/distutils/fcompiler/intel.py.bak      2009-03-29 07:24:21.000000000 -0400
+++ numpy/distutils/fcompiler/intel.py  2009-08-06 23:08:59.000000000 -0400
@@ -47,6 +47,7 @@
     module_include_switch = '-I'

     def get_flags(self):
+        return ['-fPIC', '-cm']
         v = self.get_version()
         if v >= '10.0':
             # Use -fPIC instead of -KPIC.
@@ -63,6 +64,7 @@
         return ['-O3','-unroll']

     def get_flags_arch(self):
+        return ['-xHost']
         v = self.get_version()
         opt = []
         if cpu.has_fdiv_bug():
EOF
```

Build using icc.  You have to add the `build_src` to fix a bug in the Numpy
1.3.0 distribution.

```bash
python setup.py build_src config --compiler=intel build_clib \
    --compiler=intel build_ext --compiler=intel
```

If you want to install and test without installing to the system,

```bash
python setup.py install --prefix=$PWD/d
PYTHONPATH=$PWD/d/lib/python-2.6/site-packages \
    (cd $HOME && python -c 'import scipy; scipy.test()')
```

Install as root.

```bash
sudo python setup.py install
sudo cp site.cfg /usr/local/lib/python2.6/dist-packages/numpy
```

Test (must do this outside the numpy source directory).

```bash
(cd $HOME && python -c 'import numpy; numpy.test()')
```


### SciPy 0.7.1

Now, on to SciPy.

```bash
tar zxvf scipy-0.7.1.tar.gz
cd scipy-0.7.1
```

Fix compilation with `icc`.

```diff
patch -p0 <<EOF
--- scipy/special/cephes/const.c.bak    2009-08-07 01:56:43.000000000 -0400
+++ scipy/special/cephes/const.c        2009-08-07 01:57:08.000000000 -0400
@@ -91,12 +91,12 @@
 double THPIO4 =  2.35619449019234492885;       /* 3*pi/4 */
 double TWOOPI =  6.36619772367581343075535E-1; /* 2/pi */
 #ifdef INFINITIES
-double INFINITY = 1.0/0.0;  /* 99e999; */
+double INFINITY = __builtin_inff();
 #else
 double INFINITY =  1.79769313486231570815E308;    /* 2**1024*(1-MACHEP) */
 #endif
 #ifdef NANS
-double NAN = 1.0/0.0 - 1.0/0.0;
+double NAN = __builtin_nanf("");
 #else
 double NAN = 0.0;
 #endif
EOF
```

Build.  Note that you have to set `fcompiler=intelem` for Intel 64.

```bash
python setup.py config --compiler=intel --fcompiler=intelem build_clib \
    --compiler=intel --fcompiler=intelem build_ext --compiler=intel \
    --fcompiler=intelem -I/opt/SuiteSparse/UFconfig
```

The SWIG code generates C++, but distutils doesn't use the C++ compiler to
link, so we have to do this ourselves.  There may have been a way to fix
some setup.py or something, but it's easier to just do this by hand.

```bash
for x in csr csc coo bsr dia; do
    icpc -xHost -O3 -fPIC -shared \
        build/temp.linux-x86_64-2.6/scipy/sparse/sparsetools/${x}_wrap.o \
        -o build/lib.linux-x86_64-2.6/scipy/sparse/sparsetools/_${x}.so
done
icpc -xHost -O3 -fPIC -openmp -shared \
    build/temp.linux-x86_64-2.6/scipy/interpolate/src/_interpolate.o \
    -o build/lib.linux-x86_64-2.6/scipy/interpolate/_interpolate.so
```

As with numpy, you may want to install to `$PWD/d` and test first.

```bash
python setup.py install --prefix=$PWD/d
PYTHONPATH=$PWD/d/lib/python-2.6/site-packages \
    (cd $HOME && python -c 'import scipy; scipy.test()')
```

Install as root.

```bash
sudo python setup.py install
```

Test (again, outside source directory.)

```bash
(cd $HOME && python -c 'import scipy; scipy.test()')
```
