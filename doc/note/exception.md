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

```c
#include <stdio.h>
#include <setjmp.h>
#include <signal.h>

jmp_buf buf;

void ereport(const char *errmsg)
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
            printf("catch error\n");
        }
        END_TRY();
    }
    return 0;
}

```

```text
signum: 11
catch error
```
还有一个问题，为什么只出现了一次呢？这是因为`setjmp`函数对信号的影响。如果一个信号被加入了信号屏蔽表的话，后续继续给这个
信号绑定handler是无效的。`setjmp`函数是否会对信号有影响，这个是未定义的，每个平台都不一样。
` POSIX  does  not  specify whether setjmp() will save the signal mask (to be later restored during longjmp()).`
所以我们要用另外一套函数`sigsetjmp`和`siglongjmp`。这套函数可以指定是否恢复原来的信号表。

下面看一个例子：
```c
#include <stdio.h>
#include <setjmp.h>
#include <signal.h>

sigjmp_buf buf;

void ereport(const char *errmsg)
{
    printf("ERROR: errmsg: %s\n", errmsg);
    siglongjmp(buf, 1);
}

void sig_segv_handler(int signum)
{
    printf("signum: %d\n", signum);
    siglongjmp(buf, 1);
}

#define TRY() \
    do { \
        if(sigsetjmp(buf, 1) == 0) \
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
            printf("catch error\n");
        }
        END_TRY();
    }
    return 0;
}

```

```text
signum: 11
catch error
signum: 11
catch error
```

这一次结果就正确了。这是因为我们设置了`sigsetjmp`的参数为1，这样的话会保存当前信号表，并在`siglongjmp`调用后恢复。
我们可以试试设置为0的情况，其结果和使用`setjmp`一样，不过这并不代表什么。`setjmp`在每个平台行为不一样，他对信号的行为
是未定义的。

至此，基础的东西讲完了。那么在PG中有哪些应用场景呢？

### 使用场景
+ 打断执行流，比如说到某个判断逻辑发现有问题，此时不能再继续执行下去，我们应该打个日志，然后退出执行流。并且释放资源。
比如说：
```c
/* 解析fromList
 *    解析表达式
 *      ...
 *      
 */     
```
如果在解析表达式的时候发现表达式的语法不对，此时应该退出执行流，并且释放资源。我们当然可以在解析fromList中调用解析表达式代码的下面去释放资源。
但是这样的问题是释放资源的代码会很零散。

最好的方式是业务代码里面只负责抛出异常，框架代码负责统一处理异常。

PG里面就是这么干的，在PG里面如果业务逻辑走不下去就会通过ereport来抛出ERROR异常，PG会把ERROR接住，然后统一释放资源。

比如：
```c
ereport(ERROR,
						(errcode(ERRCODE_SYNTAX_ERROR),
						 errmsg("positional argument cannot follow named argument"),
						 parser_errposition(pstate, exprLocation(arg))));
```
PG会先打印日志，然后退出当前业务流程并释放资源。

+ 如果全部异常都只有一个地方处理也不合理，我们需要在某些代码块中自己处理一些异常，比如说修改一些变量值，释放一些资源。最后交给框架的全局异常处理去释放
全局的资源。

总结来说，我们的需求有2点：
+ 全局异常处理机制
+ 局部异常处理机制
### 如何实现
+ 全局异常处理机制可以在系统启动的时候设置`sigsetjmp`函数。
+ 局部异常处理机制可以在局部函数中使用`sigsetjmp`函数，不过需要注意的是在局部异常处理的时候不需要恢复信号表，因为全局处理函数会去处理。

下面看一下PG是如何实现的：
PG在PostgresMain函数中新建了一个`local_sigjmp_buf`变量，然后注册了`sigsetjmp`并且把指针给了一个全局变量`PG_exception_stack`，这个全局
变量会被ereport函数使用。
```c
if (sigsetjmp(local_sigjmp_buf, 1) != 0)
{
    .....
}
PG_exception_stack = &local_sigjmp_buf;
```

在elog文件中的`pg_re_throw(void)`函数中调用了`siglongjmp`函数来返回当初设置的地方。
```c
if (PG_exception_stack != NULL)
    siglongjmp(*PG_exception_stack, 1);
```
如果我们在代码中使用ereport来抛出ERROR异常的话，就会通过上面的方式来达到清理资源的目的。

那么针对局部的异常处理场景，PG中提供了PG_TRY宏来处理。PG_TRY宏主要应用场景为：当要执行的代码中可能会有ERROR异常抛出，但是我们想在全局异常处理之前
先做一些处理逻辑。
比如说执行一个PLPGSQL函数的时候，我们希望该函数如果执行出错的话，先释放一些资源。
```c
PG_TRY();
	{
		/*
		 * Determine if called as function or trigger and call appropriate
		 * subhandler
		 */
		if (CALLED_AS_TRIGGER(fcinfo))
			retval = PointerGetDatum(plpgsql_exec_trigger(func,
														  (TriggerData *) fcinfo->context));
		else if (CALLED_AS_EVENT_TRIGGER(fcinfo))
		{
			plpgsql_exec_event_trigger(func,
									   (EventTriggerData *) fcinfo->context);
			retval = (Datum) 0;
		}
		else
			retval = plpgsql_exec_function(func, fcinfo, NULL, !nonatomic);
	}
	PG_CATCH();
	{
		/* Decrement use-count, restore cur_estate, and propagate error */
		func->use_count--;
		func->cur_estate = save_cur_estate;
		PG_RE_THROW();
	}
	PG_END_TRY();

```

PG中实现PG_TRY的方式为先声明一个局部变量去替换全局的`PG_exception_stack`，但是注册的时候不设置信号屏蔽表，因为在PG_RE_THROW调用后会走到全局
的异常处理，在那边会去恢复信号屏蔽表。

```c
#define PG_TRY()  \
	do { \
		sigjmp_buf *save_exception_stack = PG_exception_stack; \
		ErrorContextCallback *save_context_stack = error_context_stack; \
		sigjmp_buf local_sigjmp_buf; \
		if (sigsetjmp(local_sigjmp_buf, 0) == 0) \
		{ \
			PG_exception_stack = &local_sigjmp_buf

#define PG_CATCH()	\
		} \
		else \
		{ \
			PG_exception_stack = save_exception_stack; \
			error_context_stack = save_context_stack

#define PG_END_TRY()  \
		} \
		PG_exception_stack = save_exception_stack; \
		error_context_stack = save_context_stack; \
	} while (0)
```
