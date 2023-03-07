# cmake快速入门

## 什么是cmake

cmake是C/C++项目的Make工具,最大的优点在于cmake可以很方便的跨平台以及支持交叉编译。

cmake运行开发者编写一种平台无关的CMakeLists.txt文件来定制整个编译流程,然后根据用户的平台进一步生成本地化Makefile和工程文件。

linux平台下使用cmake生成Makefile并编译的流程如下:

1. 编写cmake配置文件CMakeLists.txt
2. 执行命令`cmake PATH`或者`ccmake PATH`生成Makefile(ccmake提供了交互式界面,当然基本没什么人用)
3. 使用make命令进行编译

本文以案例入手,一步步讲解CMake的常见用法,文中所有代码都可以在 [这里](https://github.com/helintongh/CWheel/tree/master/cmake/) 找到。

## 1.入门案例:单个源文件

本节对应源代码所在目录: [Demo1](https://github.com/helintongh/CWheel/tree/master/cmake/Demo1)

对于简单代码几行cmake代码就能搞定,假设我们的项目只有一个`main.cc`,该程序的用途是计算一个数的指数幂。


```cpp
#include <stdio.h>
#include <stdlib.h>

/**
 * power - Calculate the power of number.
 * @param base: Base value.
 * @param exponent: Exponent value.
 *
 * @return base raised to the power exponent.
 */
double power(double base, int exponent)
{
    int result = base;
    int i;

    if (exponent == 0) {
        return 1;
    }
    
    for(i = 1; i < exponent; ++i){
        result = result * base;
    }

    return result;
}

int main(int argc, char *argv[])
{
    if (argc < 3){
        printf("Usage: %s base exponent \n", argv[0]);
        return 1;
    }
    double base = atof(argv[1]);
    int exponent = atoi(argv[2]);
    double result = power(base, exponent);
    printf("%g ^ %d is %g\n", base, exponent, result);
    return 0;
}
```

### 1.1编写CMakeLists.txt

首先编写CMakeLists.txt文件,并保存在与main.cc源文件同个目录下:

```cmake
# CMake 最低版本号
cmake_minimum_required (VERSION 2.8)

# 项目信息
project (Demo1)

# 指定生成目标
add_executable(Demo main.cc)
```

CMakeLists.txt的语法比较简单,由命令,注释和空格组成,其中命令是不区分大小写的。`#`后跟的是注释,命令由命令名称,小括号和参数组成,参数之间使用空格进行间隔。

上面的CMakeLists.txt文件依次出现了三个命令:

1. `cmake_minimum_required`:指定运行此配置文件所需的CMake的最低版本;
2. `project`:参数是Demo1,表示该项目的名称是`Demo1`
3. `add_executable`:将名为main.cc的源文件编译成一个名称为Demo的可执行文件.

### 1.2编译项目

在当面目录执行`cmake .`,得到Makefile后再使用`make`命令得到Demo1可执行文件。