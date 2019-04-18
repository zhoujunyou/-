### Lua学习

背景: 由于某个公司的api对我们公司停止提供使用了  所以准备把数据从打印机里提取出来在解析。

考虑到api版本会变化 决定采用以下两种方式去进行解析适配 

1. 将获取的数据传到服务端 由服务端解析再推到客户端 好处是服务端可以随时发布 而且数据回路也会和之前一致 这样客户端也不要做什么改动。

2. 由客户端解析，要做到动态的解析 频繁发版本显然不合适。有2个方向 1个热更新 1个是配置脚本动态下发

   这里先选择了Lua 进行学习一下

本来想把活丢给服务端的。服务端也懒 我也说服不了领导 刚好自己也学习下lua 脚本 

[LuaJava在Java、Android中的使用](https://www.jianshu.com/p/12090ed4c1e2)通过这篇文章简单的理解一下原理。 



##### 什么是lua？

lua是一种可嵌入，轻量，快速，功能强大的脚本语言

```lua
--单行注释

--[[ 多行注释
多行注释--]]

--lua 关键字
--[[
and	break	do	else
elseif	end	false	for
function	if	in	local
nil	not	or	repeat
return	then	true	until
while
--]]

--申明全区变量
b=10
--删除全局变量
b=nil
```

| 数据类型 | 描述                                                         |
| -------- | ------------------------------------------------------------ |
| nil      | 这个最简单，只有值nil属于该类，表示一个无效值（在条件表达式中相当于false） |
| boolean  |                                                              |
| number   | 表示双精度类型的实浮点数                                     |
| string   | 字符串由一对双引号或单引号来表示                             |
| function | 由 C 或 Lua 编写的函数                                       |
| userdata | 表示任意存储在变量中的C数据结构                              |
| thread   | 表示执行的独立线路，用于执行协同程序                         |
| table    | Lua 中的表（table）其实是一个"关联数组"（associative arrays），数组的索引可以是数字或者是字符串。在 Lua 里，table 的创建是通过"构造表达式"来完成，最简单构造表达式是{}，用来创建一个空表 |

#### Lua变量类型

Lua 变量有三种类型：全局变量、局部变量、表中的域。

Lua 中的变量全是全局变量，那怕是语句块或是函数里，除非用 local 显式声明为局部变量。

局部变量的作用域为从声明位置开始到所在语句块结束。

变量的默认值均为 nil。

1. 避免命名冲突。 2. 访问局部变量的速度比全局变量更快。 **索引**

```lua
a = 5               -- 全局变量
local b = 5         -- 局部变量

function joke()
    c = 5           -- 全局变量
    local d = 6     -- 局部变量
end
```

#### Lua赋值语句

```lua
a， b = 10， 2*x            -->       a=10; b=2*x
x， y = y， x                     -- swap 'x' for 'y'
a， b， c = 0， 1
print(a，b，c)             --> 0   1   nil
a， b， c = 0
print(a，b，c)             --> 0   nil   nil
a， b = f() 				-->f()返回两个值，第一个赋给a，第二个赋给b

```

#### Lua循环语句

```lua
while(condition)
do
   statements
end

--var从exp1变化到exp2，每次变化以exp3为步长递增var，并执行一次"执行体"。exp3是可选的，如果不指定，默认
--为1。
for var=exp1,exp2,exp3 do   
    <执行体>  
end

repeat
   statements
until( condition )
```

#### Lua流程控制

```lua
if(布尔表达式)
then
   --[ 在布尔表达式为 true 时执行的语句 --]
end

if(布尔表达式)
then
   --[ 布尔表达式为 true 时执行该语句块 --]
else
   --[ 布尔表达式为 false 时执行该语句块 --]
end

if( 布尔表达式 1)
then
   --[ 布尔表达式 1 为 true 时执行该语句块 --]
   if(布尔表达式 2)
   then
      --[ 布尔表达式 2 为 true 时执行该语句块 --]
   end
end
```

#### Lua函数

```java
optional_function_scope function function_name( argument1, argument2, argument3..., argumentn)
    function_body
    return result_params_comma_separated
end
```

1. 函数可以作为参数传递

2. 函数可以有多个返回值

   ```lua
   > s, e = string.find("www.runoob.com", "runoob") 
   > print(s, e)
   5    10
   ```

3. 函数可以接受可变参数

   也可以通过 select("#",...) 来获取可变参数的数量

   固定参数必须放在变长参数之前

   ```lua
   function average(...)
      result = 0
      local arg={...}    --> arg 为一个表，局部变量
      for i,v in ipairs(arg) do
         result = result + v
      end
      print("总共传入 " .. #arg .. " 个数")
      return result/#arg
   end
   
   print("平均值为",average(10,5,3,4,5,6))
   ```



#### Lua运算符

```lua
a~=b --a!=b
a and b --a&&b
a or b --a||b
not --!
"a".."b" --"a"+"b"
#   --返回字符串长度 或表长度
```







1. 



