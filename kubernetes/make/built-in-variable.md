## 内置变量

make包含一些内置的常用变量。


## MAKEFLAGS


这个Flag控制make的递归调用。在多模块项目下，子系统有自己的makefile.
顶层的makefile在递归构建子模块时，`MAKEFLAGS`所包含命令行参数和flags变量会在子模块构建时一同传入。


- Makefile

```bash
# 此处给make增加 -s 选项，-s选项控制make不打印执行的命令
MAKEFLAGS += -s 

sample:
	echo $(MAKEFLAGS)
```

执行命令：

```bash
[root@localhost makefiletest]# make -k CFLAGS="-g"
sk -- CFLAGS=-g
```
### 子模块构建

增加一个子模块:

```bash
.
├── Makefile
└── subdir
    └── Makefile
```

- Makefile

```bash
subsystem:
	cd subdir && $(MAKE)
```

- subdir/Makefile

```bash
all:
	echo $(MAKEFLAGS) # 输出make命令的选项flag和接收参数
```

运行命令:

```bash
[root@localhost makefiletest]# make -sk CFLAGS="-g"
sk -- CFLAGS=-g
```


## CFLAGS


这个flag包含给C编译器的众多选项。包括debug选项，优化等级，警告等级 etc.

```bash
CC = gcc
CFLAGS = -g -Wall # Passes -g and -Wall to gcc

main.o: main.c
    $(CC) $(CFLAGS) -c main.c 
```

强制包含某个选项，但是用户又可以通过命令行传入`CFLAGS`:

```bash
CFLAGS = -g # Optional. Not required for proper compilation
ALL_CFLAGS = -I. $(CFLAGS) # -I. is required for proper compilation
main.o: main.c
        $(CC) -c $(ALL_CFLAGS) main.c
```

## CXXFLAGS  

类似`CFLAGS`,但用于C++编译器。

```bash
CXX = g++

CXXFLAGS = -g -Wall # Passes -g and -Wall to g++

main.o: main.cpp
    $(CXX) $(CXXFLAGS) -o main.o main.cpp
```

## CPPFLAGS
CPPFLAGS is used to pass extra flags to the C preprocessor. These flags are also used by any programs that use the C preprocessor, including the C, C++, and Fortran compilers. You do not need to explicitly call the C preprocessor. Pass `CPPFLAGS` to the compiler, and these will be used when the compiler invokes the preprocessor. The most common use case of `CPPFLAGS` is to include directories to the compiler search path using the -I option.

```bash
CC = gcc
CFLAGS = -g -Wall
CPPFLAGS = - I /usr/foo/bar # Search for header files in /usr/foo/bar

main.o: main.c
    $(CC) $(CPPFLAGS) $(CFLAGS) -c main.c 
```

## LDFLAGS
You can use LDFLAGS to pass extra flags to the linker `lD`. Similar to `CPPFLAGS`, these flags are automatically passed to the linker when the compiler invokes it. The most common use is to specify directories where the libraries can be found, using the `-L` option. You should not include the names of the libraries in `LDFLAGS`; instead they go into `LDLIBS`.

```bash
LDFLAGS = -L. \ # Search for libraries in the current directory
          -L/usr/foo # Search for libraries in /usr/foo

main.o: main.c
    gcc $(LDFLAGS) -c main.c 
```


翻译自：[Understanding and Using Makefile Flags](https://earthly.dev/blog/make-flags/)