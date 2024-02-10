---
title: GoogleTest简单使用
date: 2024-02-10 10:38:15
category: 
- 'C++随笔'
tags:
- 'C++'
- '单元测试'
- 'gtest'
---

# 1 Google Test简介
`Google Test`（`gtest`）是一个由`Google`提供的`C++`测试框架, 提供了丰富的断言类型和辅助函数，使得编写C++测试用例变得简单而又直观。`Google Test`旨在与`CMake`和其他构建系统无缝集成，而且与各种平台和测试工具兼容。下面是一些`Google Test`的核心特性：

# 2 Google Test基础语法
## 2.1 断言

`Google Test`提供了一系列的断言宏来检查条件是否满足。如果断言失败，测试用例被认为失败。断言分为两大类：`ASSERT_*` 和 `EXPECT_*`。

- `ASSERT_*` 版本在断言失败时会产生一个致命错误，并终止当前函数的执行。包括`ASSERT_TRUE`,`ASSERT_FALSE`, `ASSERT_EQ`等
- `EXPECT_*` 版本在断言失败时会产生一个非致命错误，当前函数会继续执行，这允许测试多个条件。包括`EXPECT_TRUE`,`EXPECT_FALSE`, `EXPECT_EQ`等

## 2.2 测试用例和测试套件

- **测试用例（Test Case）**：是指一组相关测试的集合。在`Google Test`中，使用 `TEST()` 宏来定义一个测试用例。
- **测试套件（Test Suite）**：在更早的`Google Test`版本中，测试套件是指具有相同前缀的一组测试用例的集合。在新版本中，使用 `TEST_F()` 宏来定义测试套件，并且需要定义一个测试夹具（Fixture）类。

## 2.3 测试夹具（Fixture）

测试夹具允许你为多个测试定义共享的环境或状态。你可以使用 `TEST_F()` 宏定义在同一测试夹具下的测试，所有这些测试都会共享相同的设置和清理代码。


# 3 案例: `CMake`下使用`gtest`
这个案例中, 我用自己之前学习过程中用`C++`手写常见数据结构的项目来介绍`CMake`中`gtest`的使用, 仓库在: https://github.com/ToniXWD/cppDataStructure

## 3.1 官方指导的`CMake`编写
在`CMake`中使用`gtest`不需要自行下载源码, 只需在`CMakeLists.txt`中如下编写:
```CMake
cmake_minimum_required(VERSION 3.14)
project(my_project)

# GoogleTest requires at least C++14
set(CMAKE_CXX_STANDARD 14)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

include(FetchContent)
FetchContent_Declare(
  googletest
  URL https://github.com/google/googletest/archive/03597a01ee50ed33e9dfd640b249b4be3799d395.zip
)
# For Windows: Prevent overriding the parent project's compiler/linker settings
set(gtest_force_shared_crt ON CACHE BOOL "" FORCE)
FetchContent_MakeAvailable(googletest)
```
然后就可以如下编写一个单元测试:
```cpp
// ./tests/rbTree_test.cpp
#include "../include/rbTree.hpp"
#include <algorithm>
#include <cstdlib>
#include <gtest/gtest.h>
#include <iostream>
#include <random>
#include <vector>

// Demonstrate some basic assertions.
TEST(RbTreeTest, InsertTEST) {
  // 创建 RedBlackTree 对象
  RedBlackTree<int> mySet;

  EXPECT_TRUE(mySet.empty());

  // 插入元素
  mySet.insert(42);
  mySet.insert(21);
  mySet.insert(63);
  mySet.insert(10);
  mySet.insert(4);
  mySet.insert(30);
  mySet.insert(36);
  mySet.insert(92);
  mySet.insert(75);
  mySet.insert(87);
  mySet.insert(58);

  EXPECT_TRUE(mySet.len() == mySet.getSizeByTranverse());
  EXPECT_TRUE(mySet.isBlackLenLegal());
  EXPECT_TRUE(mySet.isNoDoubleRed());
  EXPECT_EQ(mySet.len(), 11);

  mySet.insert(58);
  EXPECT_TRUE(mySet.len() == mySet.getSizeByTranverse());
  EXPECT_TRUE(mySet.isBlackLenLegal());
  EXPECT_TRUE(mySet.isNoDoubleRed());
  EXPECT_EQ(mySet.len(), 11);
}
...//省略更多的测例
```

编写完测试的`cpp`文件后, 还需要在`CMake`中进行下面的设置:
```Cmake
enable_testing()

# rbTree_test
add_executable(
  rbTree_test
  tests/rbTree_test.cpp
)
target_link_libraries(
  rbTree_test
  GTest::gtest_main
)

include(GoogleTest)
gtest_discover_tests(rbTree_test)
```
这段`CMake`是为了设置并运行名为`rbTree_test`的测试用例:
1. `enable_testing()`用于启用当前目录和以下目录中的测试功能, 使用这个命令后就可以用`make test`来运行测试。

2. `add_executable(rbTree_test tests/rbTree_test.cpp)`
   编译测试文件为可执行文件, 没啥好说的

3. `target_link_libraries(rbTree_test GTest::gtest_main)`
   将测试可执行文件链接到`gtest`的`gtest_main`库

4. `gtest_discover_tests(rbTree_test)`
    告诉`CMake`去自动发现在`rbTree_test`可执行文件中定义的所有测试用例，并创建`CTest`测试案

然后再命令行中执行:
```bash
$ mkdir build/
$ cd build/ && cmake ..
$ make
$ make test
Running tests...
Test project /home/xwd/cppDataStructure/build
    Start 1: DequeTest.Basic1
1/9 Test #1: DequeTest.Basic1 ..................................   Passed    0.00 sec
    Start 2: ListTest.Basic1
2/9 Test #2: ListTest.Basic1 ...................................   Passed    0.00 sec
    Start 3: RbTreeTest.InsertTEST
3/9 Test #3: RbTreeTest.InsertTEST .............................   Passed    0.00 sec
    Start 4: RbTreeTest.RemoveTest1
4/9 Test #4: RbTreeTest.RemoveTest1 ............................   Passed    0.00 sec
    Start 5: RbTreeTest.RemoveTest2
5/9 Test #5: RbTreeTest.RemoveTest2 ............................   Passed    0.00 sec
    Start 6: RbTreeTest.RemoveRoot
6/9 Test #6: RbTreeTest.RemoveRoot .............................   Passed    0.00 sec
    Start 7: RbTreeTest.DoubleBlackTest
7/9 Test #7: RbTreeTest.DoubleBlackTest ........................   Passed    0.00 sec
    Start 8: RbTreeTest.RemoveWithSiblingHasTwoBlackChildren
8/9 Test #8: RbTreeTest.RemoveWithSiblingHasTwoBlackChildren ...   Passed    0.00 sec
    Start 9: RbTreeTest.RandomOperation
9/9 Test #9: RbTreeTest.RandomOperation ........................   Passed    0.01 sec

100% tests passed, 0 tests failed out of 9

Total Test time (real) =   0.03 sec
```

## 3.2 自动通过`CMake`注册单元测试
之前的内容可以看出, 每个单元测试都要单独地在`CMakeLists.txt`中指定链接库等, 很繁琐, 实际上我们可以借助`CMake`的语法自动注册单元测试:
首先假设所有的单元测试都在`test`文件夹下, 且形如`*_test.cpp`, 因此可以在`test`文件夹下编写`CMake`模块:
```CMake
cmake_minimum_required(VERSION 3.10)

include(FetchContent)
FetchContent_Declare(
  googletest
  URL https://github.com/google/googletest/archive/03597a01ee50ed33e9dfd640b249b4be3799d395.zip
)

# For Windows: Prevent overriding the parent project's compiler/linker settings
set(gtest_force_shared_crt ON CACHE BOOL "" FORCE)
FetchContent_MakeAvailable(googletest)

include(GoogleTest)

enable_testing()

file(GLOB_RECURSE TEST_CPPS "${PROJECT_SOURCE_DIR}/tests/*test.cpp")

foreach (test_source ${TEST_CPPS})
    # Create a human readable name.
    get_filename_component(test_filename ${test_source} NAME)
    string(REPLACE ".cpp" "" mySTL_test_name ${test_filename})

    # Add the test target separately and as part of "make check-tests".
    add_executable(${mySTL_test_name}  ${test_source})
    target_link_libraries(${mySTL_test_name} GTest::gtest_main)


    gtest_discover_tests(${mySTL_test_name}
            EXTRA_ARGS
            --gtest_color=auto
            --gtest_output=xml:${CMAKE_BINARY_DIR}/test/${mySTL_test_name}.xml
            --gtest_catch_exceptions=0
            DISCOVERY_TIMEOUT 120
            PROPERTIES
            TIMEOUT 120
            )

    # Set test target properties and dependencies.
    set_target_properties(${mySTL_test_name}
            PROPERTIES
            RUNTIME_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/test"
            COMMAND ${mySTL_test_name}
            )
endforeach ()
```

在线获取`gtest`的部分和之前一样, 这里介绍其是如何自动发现单元测试文件的:

```cmake
file(GLOB_RECURSE TEST_CPPS "${PROJECT_SOURCE_DIR}/tests/*test.cpp")
```
`file(GLOB_RECURSE ...)`用于递归地搜索所有匹配的文件，并将它们的列表存储在变量`TEST_CPPS`中。在这种情况下，它搜索项目源目录下`tests`文件夹中所有以`test.cpp`结尾的文件。

```cmake
foreach (test_source ${TEST_CPPS})
    ...
endforeach ()
```
这个循环遍历所有找到的测试文件。对于每个文件，它执行以下操作：

- 获取文件名，去除`.cpp`后缀，创建一个易读的测试名称（`mySTL_test_name`）。
- 使用`add_executable`为每个测试文件创建一个可执行文件。
- 使用`target_link_libraries`将Google Test主库链接到每个测试可执行文件。
- 调用`gtest_discover_tests`来发现和注册测试，设置额外的参数和属性，包括输出格式（XML），是否捕获异常，测试发现超时和测试超时。
- 设置每个测试目标的属性，确保测试的可执行文件被放置在预期的目录，并指定运行测试的命令。

然后只需要在根路径下的`CMakeLists`中包含这个模块即可:
```cmake
add_subdirectory(tests)
```

之后自己新建的单元测试就可以被自动发现了