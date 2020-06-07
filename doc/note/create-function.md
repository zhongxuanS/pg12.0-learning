### 功能
PG中函数相关主要分成下面几个阶段
+ 语法分析：分析字符串，并且按照SQL创建函数语法进行判断
+ 保存到系统表：如果语法没有问题，那么检查系统表中有没有相同函数，如果有的话看是否要覆盖。
+ 函数调用解析：根据调用表达式，在系统表中找对应实现。

### 如何实现
语法相关声明都在`gram.y`文件中。
```text
CreateFunctionStmt:
			CREATE opt_or_replace FUNCTION func_name func_args_with_defaults
			RETURNS func_return createfunc_opt_list
				{
					CreateFunctionStmt *n = makeNode(CreateFunctionStmt);
					n->is_procedure = false;
					n->replace = $2;
					n->funcname = $4;
					n->parameters = $5;
					n->returnType = $7;
					n->options = $8;
					$$ = (Node *)n;
				}
			| CREATE opt_or_replace FUNCTION func_name func_args_with_defaults
			  RETURNS TABLE '(' table_func_column_list ')' createfunc_opt_list
				{
					CreateFunctionStmt *n = makeNode(CreateFunctionStmt);
					n->is_procedure = false;
					n->replace = $2;
					n->funcname = $4;
					n->parameters = mergeTableFuncParameters($5, $9);
					n->returnType = TableFuncTypeName($9);
					n->returnType->location = @7;
					n->options = $11;
					$$ = (Node *)n;
				}
			| CREATE opt_or_replace FUNCTION func_name func_args_with_defaults
			  createfunc_opt_list
				{
					CreateFunctionStmt *n = makeNode(CreateFunctionStmt);
					n->is_procedure = false;
					n->replace = $2;
					n->funcname = $4;
					n->parameters = $5;
					n->returnType = NULL;
					n->options = $6;
					$$ = (Node *)n;
				}
			| CREATE opt_or_replace PROCEDURE func_name func_args_with_defaults
			  createfunc_opt_list
				{
					CreateFunctionStmt *n = makeNode(CreateFunctionStmt);
					n->is_procedure = true;
					n->replace = $2;
					n->funcname = $4;
					n->parameters = $5;
					n->returnType = NULL;
					n->options = $6;
					$$ = (Node *)n;
				}
		;
```
在PG中，gram.y的名字虽然叫做语法文件，其实这里只是语法的一个入口，gram.y中定义了文法，相当于定义了在一个字符串中，什么样的字符串应该出现在什么地方，
组合起来应该叫做什么。但是他们之间的关系从这里是搞不定的，只能放在后面去判断。比如说returnType的类型是否存在。那么这个过程在PG中是放在`parse_analyze`
函数中进行处理的。在这个函数中会处理不同的语法元素。

具体处理创建函数的代码在`functioncmds.c`中的`CreateFunction`函数中。这个函数主要做了下面几个事情：
+ 确认语法是否正确
+ 确认是否要覆盖已有函数
+ 验证body是否语法正确
+ 保存函数依赖关系
+ 保存函数到系统表


解析函数调用过程是在`parse_func.c`中。解析函数调用过程要比创建函数更加复杂。原因有下面几个：
+ PG函数调用传参有2种方式，按照参数位置传参和按照参数名字传参，按照参数名字传参时候可以不按照声明函数的顺序来写
+ PG支持可变参数和默认参数
+ PG的数据类型有基础类型、域类型。还有隐式转换参与其中
+ PG允许有同名函数，但是参数列表不能相同
+ PG允许IN OUT INOUT三种参数模式
+ PG有namespace搜索路径，不同namespace中可以有签名相同的参数，但是搜索路径有优先级

正是因为上面这些特性，导致从一个函数调用表达式出发，要能找到想对应的匹配可能需要把上面所以情况都要考虑一遍。
具体解释在相关源码注释中。

这里提一下为什么函数签名中不包含返回值。这是因为如果包含返回值的话那么下面2个函数就应该是不同的：
```c
int test();
double test();
```

如果用户写了一个函数调用如下：
```c
test();
```
从函数调用的字符串来解析函数原型是解析不出唯一候选的。因为函数调用原型中并不包含任何关于返回值的信息。所以很多语言在设计的时候就直接让函数签名不包含
返回值。这样的话，上述2个函数是重复的不符合要求。这样就避免了这个问题。

在PG中查找函数候选的时候，就是先按照函数名去搜索所有候选，然后再逐个匹配。就是因为函数调用的字符串并不能直接得出函数原型，PG中的参数模式和传参方式都
会导致有隐藏的参数参与。