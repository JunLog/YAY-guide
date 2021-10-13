# redis

## 五种常见数据类型

### string

基本操作类型

* 添加/修改数据

> `set key value`

* 获取数据

> `get key`

* 删除数据

> `del key`

* 添加/修改多个

> `mset key1 value1 key2 value2`

* 获取多个

> `mget key1 key2`

* 获取数据字符个数

> `strlen key`

* 追加信息

> `append key value`

* 设置数据具有指定的生命周期

> setex key seconds value
>
> psetex key milliseconds value



key的设置约定

![image-20211010213846350](https://cdn.jsdelivr.net/gh/baici1/image-host/newimg/20211010213853.png)

### hash

![image-20211010214412069](https://cdn.jsdelivr.net/gh/baici1/image-host/newimg/20211010214412.png)

基本操作

* 添加/修改数据

> `hset key field value`

* 获取数据

> `hget key field`
>
> `hgetall key`

* 删除数据

> `hdel key field1 [field2]`

* 添加/修改多个

> `hmset key1 field1 value1 field2 value2`

* 获取多个

> `hmget key field1 field2`

* 获取hash表字段的数量

> `hlen key`

* 获取哈希表是否存在指定字段

> `hexists key field`

* 获取哈希表所有字段或者值

> `hkeys key`
>
> `hvals key`

### list

存储多个数据，并对数据进入存储空间顺序进行区分。

* 添加/修改数据

> `lpush key value1 [value2]` 向左添加数据
>
> `rpush key value1 [value2]` 向右边添加数据

* 获取数据

> `lrange key start stop`
>
> `lrange key 0 -1` 获取全部元素
>
> `lindex key index`
>
> `llen key`

* 获取并移除数据

> `lpop key`
>
> `rpop key `

* 规定时间获取并移除数据

> `blpop key1 timeout`

* 移除指定个数数据

> `lrem key count value`

list主要用于存储有序的集合（日志，点赞顺序等）

