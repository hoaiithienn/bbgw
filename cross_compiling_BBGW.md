- Check compiler version on BBGW (Target PC). 
```
debian@arm:~$ gcc -v

Using built-in specs.
COLLECT_GCC=gcc
COLLECT_LTO_WRAPPER=/usr/lib/gcc/arm-linux-gnueabihf/10/lto-wrapper
Target: arm-linux-gnueabihf
Configured with: ../src/configure -v --with-pkgversion='Debian 10.2.1-6' --with-bugurl=file:///usr/share/doc/gcc-10/README.Bugs --enable-languages=c,ada,c++,go,d,fortran,objc,obj-c++,m2 --prefix=/usr --with-gcc-major-version-only --program-suffix=-10 --program-prefix=arm-linux-gnueabihf- --enable-shared --enable-linker-build-id --libexecdir=/usr/lib --without-included-gettext --enable-threads=posix --libdir=/usr/lib --enable-nls --enable-bootstrap --enable-clocale=gnu --enable-libstdcxx-debug --enable-libstdcxx-time=yes --with-default-libstdcxx-abi=new --enable-gnu-unique-object --disable-libitm --disable-libquadmath --disable-libquadmath-support --enable-plugin --enable-default-pie --with-system-zlib --enable-libphobos-checking=release --with-target-system-zlib=auto --enable-objc-gc=auto --enable-multiarch --disable-sjlj-exceptions --with-arch=armv7-a --with-fpu=vfpv3-d16 --with-float=hard --with-mode=thumb --disable-werror --enable-checking=release --build=arm-linux-gnueabihf --host=arm-linux-gnueabihf --target=arm-linux-gnueabihf
Thread model: posix
Supported LTO compression algorithms: zlib zstd
gcc version 10.2.1 20210110 (Debian 10.2.1-6) 

```

- In this example, the compiler version is **10.2.1** and the target is **arm-linux-gnueabihf**.

- On Ubuntu Host PC, search for the corresponding version and download to host PC. In this example, I will download the following file **gcc-linaro-10.2.1-2021.01-x86_64_arm-linux-gnueabihf.tar.xz**
    
    https://snapshots.linaro.org/components/toolchain/binaries/10.2-2021.01-3/arm-linux-gnueabihf/gcc-linaro-10.2.1-2021.01-x86_64_arm-linux-gnueabihf.tar.xz

    (Note: Don't be confused with **gcc-arm-none-eabi-XYZ-x86_64-linux**! It is used to compile binaries for bare-metal ARM MCU!)

- Add the following lines to the bottom of **~/.profile**:

```
# set PATH so it includes toolchain for BeagleBone Black
if [ -d "/path/to/gcc-linaro-10.2.1-2021.01-x86_64_arm-linux-gnueabihf/bin" ] ; then
    PATH="path/to/gcc-linaro-10.2.1-2021.01-x86_64_arm-linux-gnueabihf/bin:$PATH"
fi

```

then run:

```
source ~/.profile
```


- Check the compiler version on Host PC: 

```
arm-linux-gnueabihf-gcc -v

Using built-in specs.
COLLECT_GCC=arm-linux-gnueabihf-gcc
COLLECT_LTO_WRAPPER=/home/thien/Documents/bbgw/gcc-linaro-10.2.1-2021.01-x86_64_arm-linux-gnueabihf/bin/../libexec/gcc/arm-linux-gnueabihf/10.2.1/lto-wrapper
Target: arm-linux-gnueabihf
Configured with: '/home/tcwg-buildslave/workspace/tcwg-dev-build_2/snapshots/gcc.git~releases~gcc-10/configure' SHELL=/bin/bash --with-mpc=/home/tcwg-buildslave/workspace/tcwg-dev-build_2/_build/builds/destdir/x86_64-pc-linux-gnu --with-mpfr=/home/tcwg-buildslave/workspace/tcwg-dev-build_2/_build/builds/destdir/x86_64-pc-linux-gnu --with-gmp=/home/tcwg-buildslave/workspace/tcwg-dev-build_2/_build/builds/destdir/x86_64-pc-linux-gnu --with-gnu-as --with-gnu-ld --disable-libmudflap --enable-lto --enable-shared --without-included-gettext --enable-nls --with-system-zlib --disable-sjlj-exceptions --enable-gnu-unique-object --enable-linker-build-id --disable-libstdcxx-pch --enable-c99 --enable-clocale=gnu --enable-libstdcxx-debug --enable-long-long --with-cloog=no --with-ppl=no --with-isl=no --disable-multilib --with-float=hard --with-fpu=vfpv3-d16 --with-mode=thumb --with-tune=cortex-a9 --with-arch=armv7-a --enable-threads=posix --enable-multiarch --enable-libstdcxx-time=yes --enable-gnu-indirect-function --with-sysroot=/home/tcwg-buildslave/workspace/tcwg-dev-build_2/_build/builds/destdir/x86_64-pc-linux-gnu/arm-linux-gnueabihf/libc --enable-checking=release --disable-bootstrap --enable-languages=c,c++,fortran,lto --prefix=/home/tcwg-buildslave/workspace/tcwg-dev-build_2/_build/builds/destdir/x86_64-pc-linux-gnu --build=x86_64-pc-linux-gnu --host=x86_64-pc-linux-gnu --target=arm-linux-gnueabihf
Thread model: posix
Supported LTO compression algorithms: zlib
gcc version 10.2.1 20210117 [releases/gcc-10 revision 05318fb8a89a166b489fa42f1f04d6c0accccc49] (GCC) 

```

- Compile a demo code:

```
arm-linux-gnueabihf-gcc -o hello hello.c
```

- Copied with scp to the Target PC and execute it

```
❯ scp hello debian@192.168.1.11:/home/debian/bin/
hello                                         100%   14KB 844.7KB/s   00:00    
❯ ssh debian@192.168.1.11
```

```
debian@arm:~/bin$ file hello
hello: ELF 32-bit LSB executable, ARM, EABI5 version 1 (SYSV), dynamically linked, interpreter /lib/ld-linux-armhf.so.3, BuildID[sha1]=3c99fabcc4c6598091df05ffeab860962accff3a, for GNU/Linux 3.2.0, with debug_info, not stripped

debian@arm:~$ ./bin/hello
Data: Hello. My name is Luong Hoai Thien!
```
