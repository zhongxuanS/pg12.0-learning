### 异常机制
C语言没有语言级别的try catch。但是很多时候代码运行的时候出现问题时我们不希望直接退出。或者提供一种类似
Java中try catch的机制。在业务代码中直接抛出对应的异常，框架会自动捕获异常并打印日志。
### 基础知识
C库函数中有2个函数`int setjmp(jmp_buf environment)`和`void longjmp(jmp_buf environment, int value)`
。

这两个函数在man手册中的描述都是叫做nolocal goto。意思就是用来做跨函数、文件和模块跳转。利用这2个函数就可以来实现
我们想要的异常。

`int setjmp(jmp_buf environment)`函数的作用是保存当前执行栈到`jmp_buf`中。初始化调用的时候返回值为0。
`void longjmp(jmp_buf environment, int value)`函数用来跳转回`setjmp`函数执行的地方。`value`用来
设置当跳转回`setjmp`的时候`setjmp`函数的返回值。

下面看一个简单的例子：
```c
#include <stdio.h>
#include <setjmp.h>

jmp_buf buf;

void ereport(const char* errmsg)
{
    printf("ERROR: errmsg: %s\n", errmsg);
    longjmp(buf, 1);
}

int main()
{
    int i = setjmp(buf);
    if(i == 0)
    {
        printf("begin execute normal code\n");
        ereport("oh throw exception");
    }
    else
    {
        printf("something wrong\n");
    }
    return 0;
}

```
```text
begin execute normal code
ERROR: errmsg: oh throw exception
something wrong

```

### 使用场景
### 如何实现