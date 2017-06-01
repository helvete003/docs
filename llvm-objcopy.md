LLVM already implements its own version of almost all of binutils. The
exceptions to this rule are objcopy and strip. This is a proposal to implement
an llvm version of objcopy/strip to complete llvm’s binutils.

Several projects only use gnu binutils because of objcopy/strip. LLVM itself
uses objcopy in fact. Chromium and Fuchsia currently use objcopy as well. If you
want to distribute your build tools this is a problem due to licensing. It’s
also a bit of a blemish on LLVM because LLVM could be made more self sufficient
if there was an llvm version of objcopy. Additionally Chromium is one of the
popular benchmarks for LLVM so it would be nice if Chromium didn’t have to use
binutils. Using
[elftoolchain](https://sourceforge.net/p/elftoolchain/wiki/Home/)
solves the licensing issue for Fuchsia but is elf specific and only solves the
issue for Fuchsia. I propose implementing llvm-objcopy to be a minimum viable
replacement for objcopy.

I’ve gone though the sources of LLVM, Clang, Chromium, and Fuchsia to try and
find the major use cases of objcopy. Here is a list of use cases I have found
and which projects use them. This list includes some use cases not found in
these 4 projects.

1. __Use Case__: Stripping debug information of an executable to a file  
   __Who uses it__: LLVM, Fuchsia, Chromium

  ```sh
  objcopy --only-keep-debug foo foo.debug
  objcopy --strip-debug foo foo
  ```
  [Example use](https://github.com/llvm-mirror/llvm/blob/cd789d8cfe12aa374e66eafc748f4fc06e149ca7/cmake/modules/AddLLVM.cmake)

  __When it is useful__:  
  This reduces the size of the file for distribution while maintaining the debug
  information in a file for later use. Anyone distributing an executable in
  anyway could benefit from this.

2. __Use Case__: Stripping debug information of a relocatable object to a file  
   __Who uses it__: None of the 4 projects considered

  ```sh
  objcopy --only-keep-debug foo.o foo.debug
  objcopy --strip-debug foo.o foo.o
  ```

  __When it is useful__:  
  In distribution of an SDK in the form of an archive it would be nice to strip
  this information. This allows debug information to be distributed separately.

3. __Use Case__: Stripping debug information of a shared library to a file  
   __Who uses it__: None of the 4 projects

   ```sh
   objcopy --only-keep-debug foo.so foo.debug
   objcopy --strip-debug foo.so foo.so
   ```

   __When is it Useful__:  
   Same benefits as the previous case. If you want to distribute a library this
   option allows you to distribute a smaller binary while maintaining the ability
   to debug.

4. __Use Case__:		Stripping an executable
   __Who uses it__:		None of the 4 projects

   ```sh
   objcopy --strip-all foo foo
   ```

   __When is it useful__:  
   Anytime an executable is being distributed and there is no reason to keep
   debugging information. This makes the executable smaller than simply
   stripping debug info and doesn't produce an extra file.

5. __Use Case__: “Complete stripping” an executable  
   __Who uses it__: None of the 4 projects

  ```sh
  eu-strip --strip-sections foo
  ```

  __When is it useful__:  
  This is an extreme form of stripping that even strips the section headers
  since they are not needed for loading. This is useful in the same contexts as
  stripping but some tools and dynamic linkers may be confused by it. This is
  possibly only valid on ELF unlike general stripping which is a valid option on
  multiple platforms.

6. __Use Case__: DWARF fission  
   __Who uses it__: Clang, Fuchsia, Chromium

  ```sh
  objcopy --extract-dwo foo foo.debug
  objcopy --strip-dwo foo foo
  ```

  [Example use 1](https://github.com/llvm-mirror/clang/blob/3efd04e48004628cfaffead00ecb1c206b0b6cb2/lib/Driver/ToolChains/CommonArgs.cpp)
  [Example use 2](https://github.com/llvm-mirror/clang/blob/a0badfbffbee71c2c757d580fc852d2124dadc5a/test/Driver/split-debug.s)

  __When is it useful__:  
  DWARF fission can be used to speed up large builds. In some cases builds can
  be too large to be handled and DWARF fission makes this manageable. DWARF
  fission is useful in almost any project of sufficient size.

7. __Use Case__: Converting an executable to binary  
   __Who uses it__: Fuchsia

   ```sh
   objcopy -O binary magenta.elf magenta.bin
   ```

   [Example use](https://fuchsia.googlesource.com/magenta/+/master/make/build.mk#20)

   __When is it useful__:  
   For kernels and embedded applications that need just the raw segments.

8. __Use Case__: Adding a gdb index  
   __Who uses it__: Chromium

  ```sh
  gdb -batch foo -ex "save gdb-index dir" -ex quit
  objcopy --add-section .gdb_index="dir/foo.gdb-index" \
           --set-section-flags .gdb_index=readonly foo foo
  ```

  [Example use](https://cs.chromium.org/chromium/src/build/gdb-add-index?type=cs&q=objcopy&l=71)

  __When is it useful__:  
  Adding a gdb index reduces startup time for debugging an application. Any
  sufficiently large program with a sufficiently large amount of debug
  information can potentially benefit from this.

9. __Use Case__: Converting between formats  
   __Who uses it__: Fuchsia (only in Magenta GCC build)

  ```sh
  objcopy --target=pei-x86-64 magenta.elf megenta.pe
  ```

  [Example use](https://fuchsia.googlesource.com/magenta/+/master/bootloader/build.mk#97)

  __When is it useful__:  
  This is primarily useful when you can’t directly target a needed format.

10. __Use Case__: Removing symbols not needed for relocation  
    __Who uses it__: Chromium

    ```sh
    objcopy --strip-unneeded foo foo
    ```

    [Example use](https://cs.chromium.org/chromium/src/third_party/libevdev/src/common.mk?type=cs&q=objcopy&l=397)

    __When is it useful__:  
    This is useful when shipping an SDK or some relocatable binaries.

11. __Use Case__: Removing local symbols  
    __Who uses it__: LLVM

    ```sh
    objcopy --discard-all foo foo
    ```

   [Example use](https://github.com/llvm-mirror/llvm/blob/cd789d8cfe12aa374e66eafc748f4fc06e149ca7/cmake/modules/AddLLVM.cmake)
   (hidden in definition of “strip_command” using strip instead of objcopy and
    using -x instead of --discard-all)

    __When is it useful__:  
    Anytime you don’t need locals for debugging this can be useful.

12. __Use Case__: Removing a specific unwanted section  
    __Who uses it__: LLVM

      ```sh
      objcopy --remove-section=.debug_aranges foo foo
      ```

    [Example use](https://github.com/llvm-mirror/llvm/blob/93e6e5414ded14bcbb233baaaa5567132fee9a0c/test/DebugInfo/Inputs/fission-ranges.cc)

    __When is it useful__:
    This is useful when you know that you have an unwanted section that isn’t
    removed by one of the other stripping options. This can also be used to
    remove an existing section for replacement by a new section.

We would like to build this up incrementally by solving specific use cases
as they come up. To start with we would like to tackle the use cases
important to us. We primarily care about fully linked executables and not
relocatable files. I plan to implement conversion from ELF to binary first.
After that I plan on implementing stripping for ELF executables.
