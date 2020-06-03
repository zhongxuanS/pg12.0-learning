### 内存管理常见问题
+ 内存泄漏
+ 野指针
+ 碎片内存太多
+ 多次向操作系统申请内存，开销太大

一般项目中如果内存申请多而且频繁的话都会使用内存池。在C++项目中有RAII和智能指针，把这两样用好的话，基本可以避免内存问题。但是在C项目中就没有这个些
好东西可以用了，只能自己撸一套。

### PG中如何解决
在PG中采用了内存上下文的机制来解决上面的问题。
+ 内存泄漏
  用内存上下文来管理一种场景下所申请的内存。当场景处理结束后手动释放。这样就不用自己去跟踪每个小内存的情况。每次需要内存的话就从对应内存上下文中申请，
  当用完后直接释放内存上下文就行。
  
+ 野指针
  这个没有好办法。
  
+ 碎片内存太多
  PG中使用了freelist来管理不用的内存。也就是说当用户主动释放内存的时候会把内存保存到freelist。用户申请内存的时候优先从freelist中找空闲内存。

+ 多次向操作系统申请内存，开销太大
  PG中会先申请一片内存，每次代码中申请内存的时候会直接从这个内存中划一片过去。省去了每次调用系统函数申请内存的过程。
  
下面看一下具体实现:
```c
typedef struct MemoryContextData
{
	NodeTag		type;			/* identifies exact kind of context */
	/* these two fields are placed here to minimize alignment wastage: */
	bool		isReset;		/* T = no space alloced since last reset */
	bool		allowInCritSection; /* allow palloc in critical section */
	const MemoryContextMethods *methods;	/* virtual function table */
	MemoryContext parent;		/* NULL if no parent (toplevel context) */
	MemoryContext firstchild;	/* head of linked list of children */
	MemoryContext prevchild;	/* previous child of same parent */
	MemoryContext nextchild;	/* next child of same parent */
	const char *name;			/* context name (just for debugging) */
	const char *ident;			/* context ID if any (just for debugging) */
	MemoryContextCallback *reset_cbs;	/* list of reset/delete callbacks */
} MemoryContextData;

typedef struct MemoryContextMethods
{
	void	   *(*alloc) (MemoryContext context, Size size);
	/* call this free_p in case someone #define's free() */
	void		(*free_p) (MemoryContext context, void *pointer);
	void	   *(*realloc) (MemoryContext context, void *pointer, Size size);
	void		(*reset) (MemoryContext context);
	void		(*delete_context) (MemoryContext context);
	Size		(*get_chunk_space) (MemoryContext context, void *pointer);
	bool		(*is_empty) (MemoryContext context);
	void		(*stats) (MemoryContext context,
						  MemoryStatsPrintFunc printfunc, void *passthru,
						  MemoryContextCounters *totals);
#ifdef MEMORY_CONTEXT_CHECKING
	void		(*check) (MemoryContext context);
#endif
} MemoryContextMethods;
```

`MemoryContextData`是内存上下文的抽象类。其中methods是虚函数，每个实现该抽象类的实现类都要实现这几个函数。
这个结构体我们主要关注`parent`、`firstChild`、`prevchild`和`nextchild`。这是因为PG中想让内存上下文有层级关系。具体表现就是一个树结构。在PG
中有一个顶级的内存上下文`TopMemoryContext`，别的内存上下文都是挂在在它下面。

在PG中实现该抽象类的实现类为：`AllocSetContext`
```c
typedef struct AllocSetContext
{
	MemoryContextData header;	/* Standard memory-context fields */
	/* Info about storage allocated in this context: */
	AllocBlock	blocks;			/* head of list of blocks in this set */
	AllocChunk	freelist[ALLOCSET_NUM_FREELISTS];	/* free chunk lists */
	/* Allocation parameters for this context: */
	Size		initBlockSize;	/* initial block size */
	Size		maxBlockSize;	/* maximum block size */
	Size		nextBlockSize;	/* next block size to allocate */
	Size		allocChunkLimit;	/* effective chunk size limit */
	AllocBlock	keeper;			/* keep this block over resets */
	/* freelist this context could be put in, or -1 if not a candidate: */
	int			freeListIndex;	/* index in context_freelists[], or -1 */
} AllocSetContext;
```
我们看到他把`MemoryContextData`放到了对象首部，这就是C语言中怎么去实现继承的方式。
`blocks`用来保存多个内存块，PG中向操作系统申请的内存都用`AllocBlock`来管理，应用中申请的内存都是在这个结构体中划一块。
`freelist`用来保存当前对象中有多少空闲内存，PG把内存按照2的次幂划分成了11个等级。从8字节开始到8192字节结束。每当有内存被释放的时候，如果被释放
内存的大小是在这个范围内，就会被保存到对应的位置上。每个空位都是一个链表，也就是说相同大小的空闲内存会以链表的方式存放。
`keeper`是用来保存哪些在重置该内存的时候不会被释放的block
`freeListIndex`因为内存上下文反复申请也很浪费，所以PG把内存上下文也做了缓存。这个值就是用来保存内存上下文的缓存位置。

每个Block中的内存用一个chunk来保存，在PG中就是`AllocChunkData`，具体结构如下：
```c
typedef struct AllocChunkData
{
	/* size is always the size of the usable space in the chunk */
	Size		size;
#ifdef MEMORY_CONTEXT_CHECKING
	/* when debugging memory usage, also store actual requested size */
	/* this is zero in a free chunk */
	Size		requested_size;

#define ALLOCCHUNK_RAWSIZE  (SIZEOF_SIZE_T * 2 + SIZEOF_VOID_P)
#else
#define ALLOCCHUNK_RAWSIZE  (SIZEOF_SIZE_T + SIZEOF_VOID_P)
#endif							/* MEMORY_CONTEXT_CHECKING */

	/* ensure proper alignment by adding padding if needed */
#if (ALLOCCHUNK_RAWSIZE % MAXIMUM_ALIGNOF) != 0
	char		padding[MAXIMUM_ALIGNOF - ALLOCCHUNK_RAWSIZE % MAXIMUM_ALIGNOF];
#endif

	/* aset is the owning aset if allocated, or the freelist link if free */
	void	   *aset;
	/* there must not be any padding to reach a MAXALIGN boundary here! */
}			AllocChunkData;
```
`size`保存的是改chunk的大小，`requested_size`保存的是这个chunk被使用了多少。`aset`比较特殊，如果这个内存片被使用的话，这里保存的是从属于那个
set，如果是空闲的话，保存的是freelist的头。

总的来说其内存的结构如图：

![image](https://github.com/zhongxuanS/pg12.0-learning/blob/master/doc/pic/f62fe885ad7504243b8d17050dd88053.png)