---
title: "Googletest with CMake"
date: 2021-01-05
draft: false
toc: false
images:
tags:
  - gtest
  - cpp
  - cmake
  - technical
---

## Background

Welcome to 2021. This is my first article for this year and I want to start with a useful tip for C++ unit testing. Unlike other languages, unit testing in C++ has never been as straightforward. However, [googletest](https://github.com/google/googletest) (also known as "gtest") has been one of the more well-received testing framework in various organizations, so that's what I will focus on.

What makes this article slightly different is that unlike the traditional way of defining an entry point for your testing program (ie. a `main()` function), I will walk through how NOT to do that - how to use `googletest` with [CMake](https://cmake.org/) without defining any entry point.

## Test case

Let's suppose that I want to test some function. The standard way to define a test case using `gtest` is the following:

```c++
// Component under test
#include <myproject/maths.h>

// gtest include
#include <gtest/gtest.h>

using namespace ::testing;

TEST(my_function_test, basic) {
    // Assume that I want to test if myproject::abs()
    // returns the correct absolute value of an integer
    ASSERT_EQ(myproject::abs(-100), 100);
}
```

As you see, I have defined a test case called `my_function_test.basic` and it tests if `abs` returns the correct absolute value - nothing exciting.

However, this is just a function - to run our test case, naturally we want to have some entry point just like any program. There's one way to do it: define a `main` function somewhere like this:

```c++
int main(int argc, char **argv) {
    ::testing::InitGoogleTest(&argc, argv);
    return RUN_ALL_TESTS();
}
```

While this works, there's a neat way to get rid of this main function completely while still being able to run our tests.

## gtest_main

A main function seems out of place being placed in a directory specifically for test cases. Fortunately, Google agrees with this idea and they've provided the `gtest_main` library that gives a basic implementation of `main()`. It means that we don't need an explicit entry point in our program.

## CMake

It's simple to use `gtest_main` with `CMake`:

```cmake
file(GLOB_RECURSE SRCS CONFIGURE_DEPENDS ${CMAKE_CURRENT_SOURCE_DIR}/*.cpp)

set(target, my_project.t)

add_executable(${target} ${SRCS})

find_package(GTest REQUIRED)
include(GoogleTest)

target_link_libraries(${target} PRIVATE
    my_project
    gmock
    GTest::GTest
    GTest::Main
)

gtest_discover_tests(${target})
```

Or it coule be simpler:

```cmake
file(GLOB_RECURSE SRCS CONFIGURE_DEPENDS ${CMAKE_CURRENT_SOURCE_DIR}/*.cpp)

set(target, my_project.t)

add_executable(${target} ${SRCS})

target_link_libraries(${target} PRIVATE
    my_project
    gmock
    gtest
    gtest_main
)

gtest_discover_tests(${target})
```

After building our project, simply run `ctest` in the build directory and all test cases will run :smiley:.
