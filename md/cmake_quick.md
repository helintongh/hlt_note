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

## 2.同一目录,多个源文件

本节对应源代码所在目录: [Demo2](https://github.com/helintongh/CWheel/tree/master/cmake/Demo2)

上面的例子只有单个源文件。现在把`power`函数单独写进一个名为MathFunctions.cc的源文件里,使得这个工程变成如下的形式:

```
./Demo2
    |
    +--- main.cc
    |
    +--- MathFunctions.cc
    |
    +--- MathFunctions.h
```

cmake改成如下形式

```cmake
# CMake 最低版本号要求
cmake_minimum_required (VERSION 2.8)

# 项目信息
project (Demo2)

# 指定生成目标
add_executable(Demo main.cc MathFunctions.cc)
```

是如果源文件很多,把所有源文件的名字都加进去将是一件烦人的工作。更省事的方法是使用 `aux_source_directory`命令,该命令会查找指定目录下的所有源文件,然后将结果存进指定变量名。其语法如下：

```cmake
aux_source_directory(<dir> <variable>)
```

可以修改cmake如下

```cmake
# CMake 最低版本号要求
cmake_minimum_required (VERSION 2.8)

# 项目信息
project (Demo2)

# 查找目录下的所有源文件
# 并将名称保存到 DIR_SRCS 变量
aux_source_directory(. DIR_SRCS)

# 指定生成目标
add_executable(Demo ${DIR_SRCS})
```

这样,CMake会将当前目录所有源文件的文件名赋值给变量DIR_SRCS,再指示变量DIR_SRCS中的源文件需要编译成一个名称为 Demo 的可执行文件.

我个人不是很推崇这种做法,容易导致cmake可读性降低。

## 3.多个目录,多个源文件

本节对应源代码所在目录: [Demo3](https://github.com/helintongh/CWheel/tree/master/cmake/Demo3)

进一步将MathFunctions.h和MathFunctions.cc文件移动到math目录下。

```
./Demo3
    |
    +--- main.cc
    |
    +--- math/
          |
          +--- MathFunctions.cc
          |
          +--- MathFunctions.h
```

对于这种情况,需要分别在项目根目录Demo3和math目录里各编写一个CMakeLists.txt文件。为了方便,可以先将math目录里的文件编译成静态库再由`main`函数调用。

根目录中的CMakeLists.txt:

```cmake
# CMake 最低版本号要求
cmake_minimum_required (VERSION 2.8)

# 项目信息
project (Demo3)
# 将源文件赋值到变量SOURCE_FILE中,可多个
set(SOURCE_FILE main.cc)

# 添加 math 子目录
add_subdirectory(math)

# 设置项目编译头文件搜索路径 -I
# 变量PROJECT_SOURCE_DIR是当前cmake的路径
include_directories(${PROJECT_SOURCE_DIR}/math)

# 指定生成目标
add_executable(Demo ${SOURCE_FILE})
# 设置要链接的库
target_link_libraries(Demo MathFunctions)
```

使用命令`add_subdirectory`指明了本项目包含一个子目录math,这样math下的CMakeLists.txt就会自动被cmake处理然后。然后使用了`include_directories`指明了依赖的头文件是在根目录的math文件夹里面。最后使用`target_link_libraries`命令指明执行文件需要链接一个`MathFunctions`的链接库。

子目录中的CMakeLists.txt:

```cmake
# 查找当前目录下的所有源文件
# 并将名称保存到 DIR_LIB_SRCS 变量
aux_source_directory(. DIR_LIB_SRCS)

# 生成链接库
add_library (MathFunctions ${DIR_LIB_SRCS})
```

### 4.怎么导入开源库

手工编译openssl为静态库,然后将其添加到文件中, 本节对应源代码所在目录: [Demo4](https://github.com/helintongh/CWheel/tree/master/cmake/Demo4)

下面是项目目录结构.

```
./Demo4
    |
    +--- main.cc
    |
    +--- math/
    |      |
    |      +--- MathFunctions.cc
    |      |
    |      +--- MathFunctions.h
    |
    +--- include/
    |      |
    |      +--- crypto/ # openssl加密库相关头文件
    |      |
    |      +--- internal/ # openssl基础库头文件
    |      |
    |      +--- openssl/ # openssl头文件合集,实际包含这个就可以了,上面两个我都删除了
    |
    |--- lib/
           |
           +--- libssl.a
           |
           +--- libcrypto.a
```

main.cc除了之前的代码还增加了一个使用openssl的例子。这里就只看主线流程

```cpp
#include <openssl/bio.h>
#include <openssl/evp.h>

// 略,一些待加密的字符串

// aes ccm加密函数
void aes_ccm_encrypt(void)
{
    // 略
}
// aes ccm解密函数
void aes_ccm_decrypt(void)
{
    // 略
}

int main()
{
    // 加密然后解密
    aes_ccm_encrypt();
    aes_ccm_decrypt();
}
```

怎么让项目找到openssl对应的头文件呢,添加cmake语句`include_directories(${PROJECT_SOURCE_DIR}/include)`,然后便是指明静态库的搜索路径以及需要链接的静态库`link_directories(${PROJECT_SOURCE_DIR}/lib)`,在`target_link_libraries`命令中添加两个库`ssl`和`crypto`.

下面是完整cmake

```cmake
# CMake 最低版本号要求
cmake_minimum_required (VERSION 2.8)

# 项目信息
project (Demo4)
# 将源文件赋值到变量SOURCE_FILE中,可多个
set(SOURCE_FILE main.cc)

# 添加 math 子目录
add_subdirectory(math)

# 设置项目编译头文件搜索路径 -I
# 变量PROJECT_SOURCE_DIR是当前cmake的路径
include_directories(${PROJECT_SOURCE_DIR}/math)
# 添加openssl的头文件搜索路径,openssl就放在include里
include_directories(${PROJECT_SOURCE_DIR}/include)

# 指定生成目标
add_executable(Demo ${SOURCE_FILE})
# 设置项目库文件搜索路径 -L
link_directories(${PROJECT_SOURCE_DIR}/lib)
# 设置要链接的库
target_link_libraries(Demo MathFunctions ssl crypto)
```

