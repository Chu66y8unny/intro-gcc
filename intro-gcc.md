# intro-gcc



How does `gcc` find the libraries? It has a built-in collection of library 
paths that are searched which can be printed by `gcc -print-search-dirs`.

You can add more library search paths by using `-L path` option.

In these paths `gcc` looks for dynamic libraries (`.so`) if available 
unless passed `-static` for static libraries (`.a`). It always add `lib` 
to the front of the name.

Program `C03/dbmain.c` requires the GNU Database Management Library (GDBM). 
I followed the steps:
```Shell
$ apt --help
$ apt search gdbm
$ apt-get --help
$ sudo apt-get install libgdbm-dev
$ dpkg --listfiles libgdbm-dev
$ ls -l /usr/include/gdbm.h
$ ls -l /usr/lib/x86_64-linux-gnu/libgdbm.a
```

Since these locations are in the search paths of `gcc` thus this command 
works:
```Shell
$ gcc -Wall -Wextra dbmain.c -lgdbm -o dbmain
```

Assume that the GDBM package is installed in `/opt/gdbm-1.8.3` then
```Shell
$ gcc -Wall -Wextra -I/opt/gdbm-1.8.3/include -L/opt/gdbm-1.8.3/lib dbmain.c -lgdbm -o dbmain
```

The options `-I` and `-L` add new directories to the beginning of the 
include path and library path respectively.

Environment variables `C_INCLUDE_PATH`, `CPLUS_INCLUDE_PATH`, 
`LIBRARY_PATH` can specify additional directories to be searched after 
`-I` and `-L` options but before the standard default directories.

To see the default include search paths:
```Shell
$ gcc -xc -E -v -     # C
$ gcc -xc++ -E -v -   # C++
$ gcc -Wp,-v -x c++ - -fsyntax-only
$ echo | gcc -E -Wp,-v -   # echo saves the user from enter Ctrl+D
$ gcc -E -Wp,-v -xc /dev/null
$ gcc -E -Wp,-v -xc++ /dev/null
$ echo | cpp -Wp,-v
$ `gcc -print-prog-name=cc1` -v
$ `gcc -print-prog-name=cc1plus` -v
```

Several directories can be specified together in an environment variable
as a colon separated list. The current directory can be specified using 
an empty path element or `.`. 

To specify multiple search path directories on the command line, the 
`-I` and `-L` can be repeated.

When environment variables and command-line options are used together 
the compiler searches the directories in the following order:
1. `-I` and `-L` options, from left to right
2. directories specified by environment variables, `C_INCLUDE_PATH`, 
   `CPLUS_INCLUDE_PATH`, `LIBRARY_PATH`
3. default system directories

## Static and Shared Libraries

External libraries are usually provided in two forms: *static libraries* 
and *shared libraries*. Static libraries are the `.a` files. When a 
program is linked against a static library, the machine code from the 
object files for any external functions used by the program is copied 
from the library into the final executable.

Shared libraries are handled with a more advanced form of linking, which 
makes the executable file smaller. They use the extension `.so`, which 
stands for shared object.

An executable file linked against a shared library contains only a small 
table of the functions it requires, instead of the complete machine code 
from the object files for the external functions. Before the executable 
file starts running, the machine code for the external functions is copied 
into memory from the shared library file on disk by the operating system
--a process referred to as *dynamic linking*.

Dynamic linking makes executable files smaller and saves disk space, 
because one copy of a library can be shared between multiple programs. 
Most operating systems also provide a virtual memory mechanism which 
allows one copy of a shared library in physical memory to be used by all 
running programs, saving memory as well as disk space.

Furthermore, shared libraries make it possible to update a library without 
recompiling the programs which use it (provided the interface to the 
library does not change).

However, when the executable file is started its loader function must find 
the shared library in order to load it into memory. By default the loader 
searches for shared libraries only in a predefined set of system 
directories, such as `/usr/local/lib` and `/usr/lib`. If the library is 
not located in one of these directories it must be added to the load path.

The simplest way to set the load path is through the environment variable 
`LD_LIBRARY_PATH`. For example, the following commands set the load path 
to `/opt/gdbm-1.8.3/lib` so that `libgdbm.so` can be found:
```Shell
$ LD_LIBRARY_PATH=/opt/gdbm-1.8.3/lib
$ export LD_LIBRARY_PATH
$ ./a.out
Storing key-value pair... done.
```

Several shared library directories can be placed in the load path, as a 
colon separated list `DIR1:DIR2:DIR3:...:DIRN`. If the load path contains 
existing entries, it can be extended using the syntax 
`LD_LIBRARY_PATH=NEWDIRS:$LD_LIBRARY_PATH`.

Note that the directory containing the shared library can, in principle, 
be stored ("hard-coded") in the executable itself using the linker option 
`-rpath` (passed from `gcc` by `-Wl,-rpath`), but this is not usually done 
since it creates problems if the library is moved or the executable is 
copied to another system.

It is possible for the system administrator to set the `LD_LIBRARY_PATH` 
variable for all users, by adding it to a default login script, such as 
`/etc/profile`. On GNU systems, a system-wide path can also be defined in 
the loader configuration file `/etc/ld.so.conf`.

```Shell
$ ldd ./a.out   # print shared object dependencies
$ export LD_TRACE_LOADED_OBJECTS=1
$ ls
$ /lib/ld-linux.so.2 --list a.out   # equivalent
$ man ld.so
$ man ld
$ man ldconfig
$ ldconfig -v | grep -v ^$'\t'   # $ syntax is for bash
                                 # see
                                 # http://www.gnu.org/software/bash/manual/bashref.html#ANSI_002dC-Quoting
                                 # https://stackoverflow.com/questions/1825552/grep-a-tab-in-unix
```

Alternatively, static linking can be forced with the `-static` option to 
`gcc` to avoid the use of shared libraries:
```Shell
$ gcc -Wall -static -I/opt/gdbm-1.8.3/include/ -L/opt/gdbm-1.8.3/lib/ dbmain.c -lgdbm
```
This creates an executable linked with the static library `libgdbm.a` 
which can be run without setting the environment variable 
`LD_LIBRARY_PATH` or putting shared libraries in the default directories.

It is also possible to link directly with individual library files by 
specifying the full path to the library on the command line.

```Shell
$ gcc -Wall -I/opt/gdbm-1.8.3/include dbmain.c /opt/gdbm-1.8.3/lib/libgdbm.a
$ gcc -Wall -I/opt/gdbm-1.8.3/include dbmain.c /opt/gdbm-1.8.3/lib/libgdbm.so   # library load path set is required to run this executable
```

`-ansi`, `-pedantic`, `-std=`

`asm` is a GNU C keyword extension but can be a valid variable name under 
the ANSI/ISO standard. We can compile `C03/ansi.c` by 
`gcc -Wall -Wextra -std=c99 ansi.c`.

```C
#include <stdio.h>

int main(void)
{
  printf("hello world\n");
  return; /* the value returned by the main function is actually the 
             return value of the printf function (the number of 
             characters printed) */
}
```

The `-Werror` option changes the default behavior by converting warnings 
into errors, stopping the compilation whenever a warning occurs.

Note that `-W` is now `-Wextra`.

# References:

- <https://stackoverflow.com/questions/3907498/how-does-a-c-compiler-find-that-lm-is-pointing-to-the-file-libm-a>
- <https://stackoverflow.com/questions/4980819/what-are-the-gcc-default-include-directories>
- <https://stackoverflow.com/questions/17939930/finding-out-what-the-gcc-include-path-is/17940271>
- <https://stackoverflow.com/questions/9922949/how-to-print-the-ldlinker-search-path/21610523>
- <https://stackoverflow.com/questions/1825552/grep-a-tab-in-unix>
- <http://www.gnu.org/software/bash/manual/bashref.html#ANSI_002dC-Quoting>
- <http://pubs.opengroup.org/onlinepubs/9699919799/basedefs/V1_chap09.html#tag_09_03>
- <https://stackoverflow.com/questions/882110/gurus-say-that-ld-library-path-is-bad-whats-the-alternative>
- <https://stackoverflow.com/questions/17914584/proper-use-of-ld-library-path-or-ldconfig-for-a-software-package>
