## 安装

**（一）解压**

```
tar -zxf lua-5.3.5.tar.gz 
```



**（二）依赖libreadline-dev**

```
yum install readline-devel
```



**（三）安装**

```
make linux test
make install
```



## 基本使用

**进入控制台**

```
lua
```



**变量**

* 全局变量

```lua
a = 1
```

* 局部变量

```lua
local a = 1
```

> 注意：lua 的变量类型是动态的，根据所附的值来确定变量的类型。
>
> 数组定义为 a = {1,2,3}



**注释**

```lua
-- 单行注释：
-- ...

-- 多行注释：
--[[
    ...
]]
```





**逻辑表达式**

```lua
-- 基本运算符
-- + - * / % -
-- 关系运算符
a == b
a ~= b
-- > < >= <=
```

> 注意：对于 1 == '1' ，lua 不会做自动转换



**逻辑运算符**

```lua
-- and/of
-- 例如： 
if a ~= 0 and a==b

-- not（逻辑非）
not(a == b)
```



**逻辑控制**

```lua
if expression then
elseif expression then
else
end

while expression do
end

-- 第一个为变量初始值，第二个为终点值，第三个为步长
for i=1,100,2 do
end
```



> 遍历数组
>
> array = {1,2,3}
>
> for i,v in ipairs(array) do
>
> end



字符串操作**

* 连接字符串（..）

```lua
a = "abcd"
b = "efg"
print(a..b)
-- 输出结果为 abcdefg
```

* 计算字符串长度（#）

```lua
a = "abcd"
b = "efg"
print(#(a..b))
-- 输出为7
```



**函数**

```lua
-- 全局函数
function add(a,b)
    return a+b
end

-- 局部函数
local function(param,par...)
    return ...
end

-- 使用函数
print(add(1,2))
```



**执行lua脚本**

```
lua demo.lua
```



## 好处

* 减少网络开销。把多个操作命令写在同一个 Lua 脚本中，Redis 去执行 Lua 脚本，使得不需要每一次命令都建立一次连接。
* 原子操作。Redis 原子化执行整个 Lua 脚本，因此使得在执行 Lua 脚本中的 Redis 命令时，不会受到干扰。
* 复用性。不需要每次调用都硬编码 Redis 命令，可以直接把整个业务流程的 Redis 命令定义为一个 Lua 脚本。



## 缺点

* 可读性比较差
* 如果lua脚本耗时，则其他客户端无法访问，直到脚本运行完。（因为原子性）



## lua 中调用 Redis 命令

```lua
redis.call('set',key,value)
```

> 注意：由于 lua 默认没有 redis 库，所有不能直接通过 lua demo.lua 执行有关 redis 的脚本操作。
>
> 可直接使用 Redis 来使用 lua 脚本（redis 内置了 lua 库）



## Redis 中使用 lua 脚本

在 redis 中使用 `eval "script" keynums key [key...] arg [arg...]`

例如

```
eval "redis.call('set','name','cavielee')"
```



注意：可以在脚本中通过 KEYS[index] 和 ARGV[index] 去获得相应的参数。（index 从1开始，一定要大写）

```

```



demo.lua

```lua

```



执行外置 lua 脚本

逗号隔开key 和 arg，并且要有空格

```
./redis-cli --eval demo.lua 127.0.0.1 , 60 3
```



注意：

当出现死循环时，则其他客户端无法访问该 redis，可以通过 kill 掉进程或者 SHUTDOWN NOSAVE



### redisTemplate中使用 lua

```java
/* 接口
public <T> T execute(RedisScript<T> script, List<K> keys, Object... args) {
    return scriptExecutor.execute(script, keys, args);
}
脚本封装类
static <T> RedisScript of(String script, Class<T> resultType) {
	...
}
*/
(String) stringRedisTemplate.execute(RedisScript.of("return redis.call('get','b')", String.class),
				new ArrayList<String>(), "b")
```



改进：由于每一次都反复的写大量的 Script（String），因此可以把该脚本保存在 redis 中，并返回一个 sha（摘要），下次可直接通过摘要来确定要执行哪一个 lua 脚本。

```java
// 获取该脚本的sha1
RedisScript.of("return redis.call('get','b').getSha1();
```

可直接通过redis-cli 执行

```
evalsha sha
```

在 RedisTemplate 中执行

```java
RedisConnection redisConnection = stringRedisTemplate.getConnectionFactory().getConnection();

/* 接口
evalSha(String scriptSha, ReturnType returnType, int numKeys, byte[]... keysAndArgs);
*/
Object object = redisConnection.evalSha("eefb072c762647cc8d8a40e7fe048aaf8872d416", ReturnType.VALUE, 0,new byte[0]);
System.out.println(new String((byte[])object));

```



