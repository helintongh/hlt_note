## 一文搞懂Shell编程

文章内容:

[]()

## Linux Shell 简介
Linux shell是一种特殊的交互式工具。核心是命令行提示符。命令行提示符是shell负责 交互的部分。它允许你输入文本命令，然后解释命令，并在内核中执行。
shell包含了一组内部命令，用这些命令可以完成诸如文件操作、进程操作、用户管理等操作。
将多条shell命令写入一个.sh文件中执行------Shell脚本，文件后缀.sh。
默认使用的shell 是bash，shell是解释型语言。 内部命令==shell自带的命令。

```bash
#!/bin/bash   脚本的声明： 我要使用哪个shell解释器

#this is my first script!    在shell中“#”表示注释信息
echo "Hello World!0"
echo "Hello World!1"
echo "Hello World!2"
echo "Hello World!3"
```
脚本的执行

- sh -x hello.sh    (便于排查错误)
- chmod +x hello.sh ; ./hello.sh  

本文不是介绍命令的,而是介绍Shell这门语言。

## Shell的逻辑运算符

- 重定向输入输出

`>`是输入

```bash
echo "123456" > a.txt
cat a.txt
# a.txt的内容为123456
```

`<`是输出
```bash
cat < /etc/passwd > a.txt # 把/etc/passwd中的数据重定向到a.txt中
```

`--stdin`

```bash
echo "12345678" |  passwd  --stdin root
```



`>>` 表示追加 ， `>`表示覆盖内容

```bash
echo 123 > a.txt
cat a.txt # 输出结果为123

echo 123 >> a.txt
cat a.txt 
# 多行注释
<< EOF
输出结果为:
123
123
EOF
```

`|`管道

著名使用方法:

```bash
ps -ef | grep xxx # 查找名字是xxx的进程
```

`&&`逻辑与

```bash
ls && ls -l
```

`||`逻辑或

```bash
ls || ls -l
```

`[ ]` 条件判断,其中`-d` 检测目录,`-f`检测文件

```bash
[ -d /root ] && echo "have root" # 注意-d和要检测的路径不能紧贴[]符号
# 当存在/root目录时,输出have root
```
## Shell环境变量

Shell的变量有全局变量与局部变量。需要掌握 定义环境变量,删除环境变量,PATH环境变量,数组变量。

bash shell用一个叫作环境变量(environment variable)的特性来存储有关shell会话和工作环境的信息(这也是它们被称作环境变量的原因)。这项特性允许你在内存中存储数据，以便程序或shell中运行的脚本能够轻松访问到它们。这也是存储持久数据的一种简便方法。

在shell编程中尽量使用大写字符作为变量名称。

变量名称=变量值

`echo $变量名称`

变量名称不能以数字开头

```bash
1a = 1 # 报错
```

- **全局变量与局部变量**

全局变量: 所有shell中生效 `export a=1`

```bash
echo $b
b=2 # 等号别分开
export b # 声明为全局,存放到环境变量中
bash # 进入一个新的终端
echo $b # 输出为2
```


局部变量
```bash
b=2
```

查看系统中的环境变量

```bash
env
```

- 删除**环境变量**

查找上面声明的b,并删除它
```bash
env | grep b=2 # 输出为b=2
unset b 	   # 删除环境变量b
env | grep b=2 # 输出为空
```

环境变量还可以通过磁盘文件 `/etc/profile` 来持久化。

- **数组变量**

这里数组下标从1开始还是从0开始应该与bash解释器有关,当我使用zsh时是1为下标。所以这里有一个坑点生产环境尽量别用zsh。
```bash
students=(aa bb cc dd)
echo $students		# 输出为aa
echo ${students}	# 输出为aa,不推荐前两种写法隐式不好。
echo ${students[0]} # 输出为aa
echo ${students[1]} # 输出为bb
echo ${students[2]} # 输出为cc
echo ${students[3]} # 输出为dd
```

## IF语句

- IF THEN
- IF ELSE
- IF ELSE ELIF
- 嵌套IF

一行实现多条命令

```bash
echo 1; echo 2
```

- **IF THEN**

```bash
if 条件1
  then
     开始执行条件1对应的任务
fi
```

```bash
#!/bin/bash
NAME=test

if [ $NAME == 'test' ]
  then
    echo $NAME
fi
```

- **IF ELSE**

如果条件不成立则执行else中的语句。

```bash
if 条件
  then 
     执行条件对应的语句
  else
     条件不成立执行的语句
fi
```

```bash
#!/bin/bash
NAME=test1

if [ $NAME == 'test' ]
  then
    echo $NAME
  else
    echo "error!"
fi

```


- **IF ELIF ELSE**

通过elif then来定义多个条件匹配。

```bash
if 条件1
  then 
     条件1语句
  elif 条件2
    then
      条件2 语句
  elif 条件3
    then
      条件3 语句
  else
    上述条件都不成立的语句
fi
```

```bash
#!/bin/bash
NAME=test1

if [ $NAME == 'test' ];then
    echo $NAME
elif [ $NAME == 'test2' ];then
  echo "my name is $NAME"
elif [ $NAME == 'test1' ];then
  echo "my name is test1  heihei~"
else
  echo "error!"
fi
```


- **嵌套IF语句**

if语句中包含if语句

```bash
#!/bin/bash
NAME=test
AGE=20

if [ $NAME == 'test' ];then
    echo $NAME
    if [ $AGE -lt 30 ]; then
      echo "$NAME < 30"
    else
      echo "$NAME > 30"
    fi
fi
```

扩展比较运算符

```bash
-eq 等于
-lt 小于
-bt 大于
```

## CASE语句

```bash
case 变量 in
	method01)
		dosomething sh
		;;
	method02)
		dosomething sh
		;;
	*)
		default sh
		;;
esac
```

- 实例之服务脚本

```bash
OPTION="start"

case "$OPTION" in 
  start)
    echo "server starting......."
    ;;
  stop)
    echo "server stoping....."
    ;;
  restart|reload)
    echo "server stoping....."
    echo "server stoped....."
    echo "server starting....."
    echo "server started......"
    ;;
  *)
    echo "sh xxx.sh start|stop|restart|reload|"
    ;;
esac
```

- 位置变量

$1 $2 $3....， $0 是脚本本身，后面都是参数，多个参数需要通过空格分割。
这个很好理解就是C语言`int main(int argc, char *argv[])`中的argv参数。
```bash
#!/bin/bash
case "$1" in 
  start)
    echo "server starting......."
    ;;
  stop)
    echo "server stoping....."
    ;;
  restart|reload)
    echo "server stoping....."
    echo "server stoped....."
    echo "server starting....."
    echo "server started......"
    ;;
  *)
    echo "sh xxx.sh start|stop|restart|reload|"
    ;;
esac
```

使用上述脚本如下:

```bash
sh location.sh start
# 输出server starting.......
```

## FOR循环

```bash
for i in 1 2 3   ### list 列表  数组 遍历
do
  echo $i
done
```

```bash
for f in `ls` # 执行ls
do 
  echo $f  	  # 挨个遍历当前路径下的文件名并打印
done
```

`$()` 获取命令的执行结果

- 实例: 批量复制文件

```bash
for dir in opt tmp
do
  touch /$dir/test.txt
done
```

遍历

```bash
for (i=10; i<20; i++)
do
  echo $i
done
```

- 跳出**循环**

continue: 跳出本次循环,继续下次循环
break: 跳出循环。

```bash
for name in zhangsan lisi wangwu 
do 
   if [ $name == 'zhangsan' ]
     then
       echo "zhangsan"
       continue;
     else
       echo $name
   fi
done
```

输出全部姓名。

```bash
for name in zhangsan lisi wangwu
do
	if [ $name == 'zhangsan' ]
	  then
	  	echo "zhangsan"
	  	break;
	  else
	  	echo $name
	fi
done
```
只输出zhangsan

## WHILE循环语句

当条件成立后执行循环，直到条件发生变化。

```bash
while true # 死循环
do
	echo "hello"
done
```

true : 条件成立, false: 条件不成立


**扩展递增**

```bash
NUM=`expr $NUM + 1`
 ((NUM++))
```

**数值运算** expr

```bash
expr 1 + 1
expr 1 \* 2
expr 6 / 2
expr 1 - 1 

```

**猜数字**

shell交互指令,通过read进行交互。理解为C++的cin,C语言的scanf

```bash
#!/bin/bash
read -p "Please enter an  number:" NUM


while [ $NUM -lt 60 ]
do 
   echo "Your score is down!"
   break;
done
```
上述代码要放在脚本文件里。


```bash
#!/bin/bash
while true
do 
   read -p "Please enter an  number:" NUM
   if [ $NUM -lt 60 ];then
     echo "Your score is down!"
   elif [ $NUM -gt 70 ];then
     echo "Your score is good!"
   elif [ $NUM -eq 100 ]; then
     echo "Your score is very good!"
   fi  
done
```
上述代码亦要放在脚本文件里。

## awk和sed处理

文本处理还有两大利器awk和sed。

awk可以参照大神左耳朵耗子的文章 [awk简明教程](https://coolshell.cn/articles/9070.html)

我这篇文章只是为了讲述shell语言不是讲shell生态中的放放面面。而且awk和sed需要就可以去查文档即可。

## Shell函数

函数声明

```bash
函数名称(){
	echo "Hello World!"
}

函数名称 # 直接输入函数名称就调用该函数了
```

```bash
#!/bin/bash
hello(){
   echo "Hello World!"
}

hello
```

- **实现一个服务器启动的mock脚本**

```bash
#!/bin/bash

start(){
	echo "start services..."
	# systemctl start nginx
}

stop(){
	echo "stop services..."
}
# restart函数执行stop函数然后再执行start函数
restart(){
	stop
	start
}

case $1 in
	start)
		start
		;;
	stop)
		stop
		;;
	restart)
		restart
		;;
	*)
		echo "please sh xxx.sh start|stop|restart"
		;;
esac
```