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

   string是redis最基本的类型,一个是key