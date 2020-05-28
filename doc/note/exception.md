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

在执行完`longjmp`后，又回到了`setjmp`函数，并且`setjmp`函数返回值是执行`longjmp`函数
的`value`参数。

这样的话我们可以用这个机制来模拟一把`try...catch`。

```c
#include <stdio.h>
#include <setjmp.h>

jmp_buf buf;

void ereport(const char* errmsg)
{
    printf("ERROR: errmsg: %s\n", errmsg);
    longjmp(buf, 1);
}

#define TRY() \
    do { \
        if(setjmp(buf) == 0) \
        {
#define CATCH() \
        } \
        else \
        {
#define END_TRY() \
        } \
    } while(0)

int main()
{
    TRY()
    {
        // dosomethind()
        // throw error
        ereport("somethind wrong!!");
    }
    CATCH()
    {
        printf("catch error");
    }
    END_TRY();
    return 0;
}

```

```text
ERROR: errmsg: somethind wrong!!
catch error
```

是不是有几分神似了。但是使用这种方式只能catch到调用`longjmp`的场景，代码里面如果出现了段错误怎么办？
这个时候就要用到Linux里面的信号机制了。

我们在try代码里面给`SIGSEGV`安装上一个handler。在这个handler里面调用`longjump`，这样就可以把段错误
给catch住。

下面看一个例子：
```c
#include <stdio.h>
#include <setjmp.h>
#include <signal.h>

jmp_buf buf;

void ereport(const char* errmsg)
{
    printf("ERROR: errmsg: %s\n", errmsg);
    longjmp(buf, 1);
}

void sig_segv_handler(int signum)
{
    printf("signum: %d\n", signum);
}

#define TRY() \
    do { \
        if(setjmp(buf) == 0) \
        {
#define CATCH() \
        } \
        else \
        {
#define END_TRY() \
        } \
    } while(0)

int main()
{
    TRY()
    {
        signal(SIGSEGV, sig_segv_handler);
        int *p = NULL;
        *p = 1;
    }
    CATCH()
    {
        printf("catch error");
    }
    END_TRY();
    return 0;
}

```

```text
signum: 11
signum: 11
....
```
一直会输出`signum: 11`。这是因为在发生段错误的时候会跳入对应的handler，当handler函数结束后，又会回到发生段错误
的地方继续执行。这样的话就又会发生段错误。显然，我们的目的是不能再回到段错误的地方执行了，要跳到catch里面。
这时候就可以在handler里面调用`longjmp`函数就可以了。

```c
#include <stdio.h>
#include <setjmp.h>
#include <signal.h>

jmp_buf buf;

void ereport(const char* errmsg)
{
    printf("ERROR: errmsg: %s\n", errmsg);
    longjmp(buf, 1);
}

void sig_segv_handler(int signum)
{
    printf("signum: %d\n", signum);
    longjmp(buf, 1);
}

#define TRY() \
    do { \
        if(setjmp(buf) == 0) \
        {
#define CATCH() \
        } \
        else \
        {
#define END_TRY() \
        } \
    } while(0)

int main()
{
    TRY()
    {
        signal(SIGSEGV, sig_segv_handler);
        int *p = NULL;
        *p = 1;
    }
    CATCH()
    {
        printf("catch error");
    }
    END_TRY();
    return 0;
}

```

```text
signum: 11
catch error
```

但是上面的例子还有有问题，我们来看一个场景：
```c
#include <stdio.h>
#include <setjmp.h>
#include <signal.h>

jmp_buf buf;

void ereport(const char* errmsg)
{
    printf("ERROR: errmsg: %s\n", errmsg);
    longjmp(buf, 1);
}

void sig_segv_handler(int signum)
{
    printf("signum: %d\n", signum);
    longjmp(buf, 1);
}

#define TRY() \
    do { \
        if(setjmp(buf) == 0) \
        {
#define CATCH() \
        } \
        else \
        {
#define END_TRY() \
        } \
    } while(0)

int main()
{
    for (int i = 0; i < 2; ++i)
    {
        TRY()
        {
            signal(SIGSEGV, sig_segv_handler);
            int *p = NULL;
            *p = 1;
        }
        CATCH()
        {
            printf("catch error");
        }
        END_TRY();
    }
    return 0;
}

```

```text
signum: 11

Process finished with exit code 139 (interrupted by signal 11: SIGSEGV)
```

好像失效了。并没有catch到错误。但是应该输出一次`catch error`才对，这是为什么呢？这是因为
printf没有加上`\n`。这会导致字符串还在buffer中，没有强制刷新到终端显示。

### 使用场景
### 如何实现