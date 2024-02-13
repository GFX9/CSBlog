---
title: 'Google Test 简单使用'
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

### 2.2.1 测试用例
**测试用例（Test Case）**：是指一组相关测试的集合。在`Google Test`中，使用 `TEST()` 宏来定义一个不需要额外的设置或清理过程的简单测试用例。使用 `TEST` 时，只需提供测试案例名称和测试名称，然后编写测试代码块。案例如下:
```cpp
TEST(RbTreeTest, InsertTEST) {
  // 测试代码在这里
  EXPECT_EQ(1, 1); // 一个示例断言
}
```
其中`TEST`宏后的2个参数唯一标记了一个测试用例, 第一个参数可以重复, 但2个参数不能同时重复。
使用 `TEST`，每个测试是独立的，测试之间不共享任何状态。这个宏适合于无状态的测试，或者不需要为多个测试维护一个共同的环境时。

### 2.2.2 测试套件
**测试套件（Test Suite）**：在更早的`Google Test`版本中，测试套件是指具有相同前缀的一组测试用例的集合。在新版本中，使用 `TEST_F()` 宏来定义测试套件，并且需要定义一个测试固件（`Fixture`）类。

`TEST_F` 宏在有一个测试固件时使用，测试固件是一种用来重用相同的设置和清理代码为多个测试服务的方法。测试固件通过一个从 `::testing::Test` 派生的类来定义。然后，可以重写 `SetUp` 和 `TearDown` 方法来初始化和清理测试环境。

下面是一个使用 `TEST_F` 的例子：

```cpp
// 定义测试固件
class MyTestFixture : public ::testing::Test {
protected:
  void SetUp() override {
    // 设置测试环境的代码
  }

  void TearDown() override {
    // 清理测试环境的代码
  }
};

// 使用 TEST_F 编写使用测试固件的测试
TEST_F(MyTestFixture, 测试名称) {
  // 可以使用设置好的环境的测试代码
  EXPECT_EQ(1, 1); // 一个示例断言
}
```

使用 `TEST_F`，同一个固件内的每个测试按照它们定义的顺序运行，但`Google Test`确保每个测试是隔离的；也就是说，在每个测试之前，环境都会重置为通过 `SetUp` 建立的初始状态。这样，一个测试所做的改变不会影响到另一个测试。


# 3 案例: `CMake`下使用`gtest`
这个案例中, 我用自己之前学习过程中用`C++`手写常见数据结构的项目来介绍`CMake`中`gtest`的使用, 仓库在: https://github.com/ToniXWD/cppDataStructure

## 3.1 简单测试用例
### 3.1.1 官方指导的`CMake`编写
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

### 3.1.2 自动通过`CMake`注册单元测试
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

## 3.2 测试套件
如果有这样一种情况, 多个测试用例中, 代码初始化部分逻辑是相同的, 可以将其设置为测试套件, 下面是一个测试堆(`Heap`)数据结构的单元测试:
```cpp
#include "../include/heap.hpp"
#include <gtest/gtest.h>
#include <iostream>

// 定义测试固件
class InitHeap : public ::testing::Test {
protected:
  Heap<int> minHeap;
  void SetUp() override {
    // 设置测试环境的代码
  }

  void TearDown() override {
    // 清理测试环境的代码
  }
};

// Test case for inserting elements into the heap
TEST_F(InitHeap, InsertAndSize) {
  minHeap.insert(5);
  minHeap.insert(2);
  minHeap.insert(8);

  EXPECT_EQ(minHeap.size(), 3);
}

// Test case for removing the root element
TEST_F(InitHeap, RemoveRoot) {
  minHeap.insert(5);
  minHeap.insert(2);
  minHeap.insert(8);

  EXPECT_EQ(minHeap.removeRoot(), 2); // Assuming a min heap
  EXPECT_EQ(minHeap.size(), 2);
}

// Test case to check heap property is maintained after insertions
TEST_F(InitHeap, HeapPropertyAfterInsertion) {
  minHeap.insert(5);
  minHeap.insert(2);
  minHeap.insert(8);
  minHeap.insert(1);
  minHeap.insert(3);

  EXPECT_EQ(minHeap.removeRoot(), 1);
  EXPECT_EQ(minHeap.removeRoot(), 2);
  EXPECT_EQ(minHeap.removeRoot(), 3);
  EXPECT_EQ(minHeap.removeRoot(), 5);
  EXPECT_EQ(minHeap.removeRoot(), 8);
}

// Test case for handling removal from an empty heap
TEST_F(InitHeap, RemoveFromEmptyHeap) {
  EXPECT_THROW(minHeap.removeRoot(), std::out_of_range);
}

// Test case for dynamic resizing of the heap
TEST_F(InitHeap, ResizeHeap) {
  size_t initialCapacity = 32; // Assuming initial capacity is 32
  for (int i = 0; i < initialCapacity + 1; ++i) {
    minHeap.insert(i);
  }

  EXPECT_GT(minHeap.size(), initialCapacity);
  for (int i = 0; i <= static_cast<int>(initialCapacity); ++i) {
    EXPECT_EQ(minHeap.removeRoot(), i); // Assuming a min heap
  }
}

// Test case for max heap property
TEST(HeapTest, MaxHeapProperty) {
  Heap<int, std::greater<int>> maxHeap;
  maxHeap.insert(5);
  maxHeap.insert(2);
  maxHeap.insert(8);
  maxHeap.insert(1);
  maxHeap.insert(3);
  maxHeap.insert(26);
  maxHeap.insert(-5);

  EXPECT_EQ(maxHeap.removeRoot(), 26);
  EXPECT_EQ(maxHeap.removeRoot(), 8);
  EXPECT_EQ(maxHeap.removeRoot(), 5);
  EXPECT_EQ(maxHeap.removeRoot(), 3);
  EXPECT_EQ(maxHeap.removeRoot(), 2);
  EXPECT_EQ(maxHeap.removeRoot(), 1);
  EXPECT_EQ(maxHeap.removeRoot(), -5);
}
```
除了最后一个测试用例外, 每个测试都是初始化一个泛型为`int`的小根堆, 因此可以定义一个继承自`testing::Test`的类`InitHeap`, 并添加一个成员变量`minHeap`, 此后在使用`TEST_F`宏的测试案中将第一个参数设置为`InitHeap`, 然后就可以不用初始化小根堆了。如果有别的需求， 还可以重写`TearDown`和`SetUp`成员方法