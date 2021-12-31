### redis安装

```bash
sudo apt install redis-server
```



### redis配置文件

/etc/redis/redis.conf

下面是redis的配置文件,比较重要的字段

```
# 开启守护进程
# 当redis作为守护进程运行的时候,它会写一个pid到/var/run/redis.pid文件里面
daemonize on

# 监听端口号,默认是6379，如果设为0。那么redis将不在socket上监听任何客户端连接。
port 6379

# 设计数据库的数目
databases 16

# 根据给定时间间隔和写入次数将数据保存到磁盘
save 900 1 			# 900s内至少有1个key的值变化,则保存
save 300 10	
save 60 10000

# 绑定的地址
bind 127.0.0.1 ::1 # 代表ipv4的127.0.0.1以及ipv6的本地地址，为此时代表只允许本地连接
```

### redis的简单使用

连接redis

```bash
redis-cli # 进入redis客户端
```

| select $number | 切换数据库号              |
| -------------- | ------------------------- |
| DBSIZE         | 查看当前数据库的key数量   |
| keys *         | 查看key的内容             |
| FLUSHDB        | 清空当前数据库的key的数量 |
| FLUSHALL       | 清空所有库的key(慎用)     |
| exists key     | 判断数据库是否存在        |



```shell
127.0.0.1:6379> select 2 # 切换数据库2
127.0.0.1:6379[2]> select 26 # 会报错,因为默认只有16个数据库。index为0-15
```

### redis常用数据类型



#### reids-string

1. string(字符串)

   string是redis最基本的类型,一个key对应一个string

   string可以包含任何数据,最大不能超过512M

   

   redis操作string有如下基本操作

   1. set/get/append/strlen

      - set 设置值
      - get 获取值
      - mset 设置多个值
      - mset 获取多个值
      - append 添加字段
      - del 删除
      - strlen 返回字符串长度

   2. incr/decr/incrby/decrby

      - incr 增加1，注增加不能增加字符串
      - decr 减少1
      - incrby 指定增加多少
      - decrby 指定减少多少

   3. getrange/setrange

      - getrange 获取指定区间范围的值,类似于between ... and的关系
      - setrange 代表从第几位开始替换,下标是从0开始的

      0到-1代表全部

      ```bash
      set k1 helintongh
      getrange k1 0 -1 # "helintongh"
      getrange k1 0 2  # "hel"
      setrange k1 0 xxx # xxxintongh
      ```

      

   下面看一个实际操作的例子。

   ```bash
   127.0.0.1:6379> set name hlt # 设置name为key,value是hlt
   OK
   127.0.0.1:6379> get hlt
   (nil)
   127.0.0.1:6379> get name # 获取key name对应的value
   "hlt"
   127.0.0.1:6379> set k1 v1
   OK
   127.0.0.1:6379> get k1
   "v1"
   127.0.0.1:6379> mset k2 v2 k3 v3 
   OK
   127.0.0.1:6379> get k2
   "v2"
   127.0.0.1:6379> get k3
   "v3"
   127.0.0.1:6379> get v2
   (nil)
   127.0.0.1:6379> APPEND name xxx # 在name这个key对应的value中增加xxx字符
   (integer) 6
   127.0.0.1:6379> get name
   "hltxxx"
   127.0.0.1:6379> del k1 # 删除k1
   (integer) 1
   127.0.0.1:6379> get k1
   (nil)
   127.0.0.1:6379> strlen name # 获取name的长度
   (integer) 6
   127.0.0.1:6379> set num 1
   OK
   127.0.0.1:6379> incr num
   (integer) 2
   127.0.0.1:6379> get num
   "2"
   127.0.0.1:6379> incr num
   (integer) 3
   127.0.0.1:6379> decr num
   (integer) 2
   127.0.0.1:6379> get num
   "2"
   127.0.0.1:6379> incrby num 2 # num对应的value增加2
   (integer) 4
   127.0.0.1:6379> decrby num 3 # num对应的value减少3
   (integer) 1
   127.0.0.1:6379> 
   ```

   