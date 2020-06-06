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

### 总结