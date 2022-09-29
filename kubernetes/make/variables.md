本章记录下make[怎么使用变量](https://ftp.gnu.org/old-gnu/Manuals/make-3.79.1/html_chapter/make_6.html) 。

>  makefile广泛用于各种类型的项目（包括kubernetes），还是要学习下，不然真的跟看天书一样

## 两种赋值方式

1.  递归扩展式  

递归式扩展的意思是，这种变量使用`=`或者`define`赋值，赋值运算符右边如果包含其他变量，这些变量也会被解析并进行插值。

比如下面例子，输出为`Huh?`
```bash
foo = $(bar)
bar = $(ugh)
ugh = Huh?

all:;echo $(foo)
```

解析链为：
```bash
$(foo) => $(bar) = >$(ugh) => Huh?
```
也就是说只要赋值运算符右边有其他变量，这些变量就会被解析，直到全部扩展为普通文本，这种赋值方式称为为递归式扩展（recursive expansion）。


2. 简单扩展

简单扩展使用的赋值符号是`:=`.

简单扩展只能进行简单插值，如下所示：

```bash
x := foo
y := $(x) bar

hello:
	echo $(x)
```

输出为：
```bash
echo foo bar
foo bar
```

以下两个例子可以解释直接扩展的插值时机为赋值动作发生时，而不是输出时。

- 例子A

```bash
x := $(t) # x=''
y := $(x) bar  # y=bar
t := sfd

hello:
	echo $(y)
```

- 例子B

```bash
t := foo
x := $(t)   # x=foo 
y := $(x) bar # y= foo bar

hello:
	echo $(y) 
```

那如果把例子A里的赋值符改成`=`,则会输出`y=foo bar`


## 条件赋值

条件赋值符(onditional variable assignment operator)为`?=`

比如下面这个例子，取消第一行注释，输出为`hello`，不取消则输出为`bar`
```bash
# FOO = hello
FOO ?= bar # 如果未定义则赋值为bar

hello:
	echo $(FOO)
```

条件赋值符相当于一种语法糖，复杂写法相当于：

```bash
# FOO = hello
ifeq ($(origin FOO), undefined)
  FOO = bar
endif

hello:
	echo $(FOO)
```

## 替换插值
替换插值很好理解，就是字符串替换。

```bash
foo := a.o b.o c.o
bar := $(foo:.o=.c)
hello:
	echo $(bar)  # => a.c b.c c.c
```
`$(var: a=b)`这种语法会将子串a替换为子串b.


## 追加插值

追加插值是在字符串后追加字符串。

```bash
foo = hello
foo += -s 

sample:
	echo $(foo) # => foo = hello -s
```


## define

定义多行变量

```bash
all:                    
    @$(cmd)

define cmd
    echo "test define 1"
    echo "test define 2"
    echo "test define 3"
endef
```

> `@`符号表示不回显命令本身.

输出为：

```bash
test define 1
test define 2
test define 3
```