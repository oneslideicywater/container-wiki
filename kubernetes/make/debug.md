
## Debug Make

`warning`函数用于debug makefile,几乎可以任意位置放置。

```bash
sample:
   $(warning debug info here at $(shell date))
```

输出为:

```bash
Makefile:3: debug info here at Thu Sep 29 10:53:48 CST 2022
make: Nothing to be done for `hello'.
```