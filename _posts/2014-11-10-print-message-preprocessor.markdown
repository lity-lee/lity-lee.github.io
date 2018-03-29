---
layout: post
category: linux
title:  "print a message from the C preprocessor"
tags: [gcc,编译,编程]
---

预编译过程中，经常遇到宏的定义和代码分支处理，特别是在接触一份新的代码的时候。在C/C++语言开发过程中，遇到代码修改了很多次却一点作用都没有，很可能是哪里宏控制起作用导致的，不防这样来定位一下。

<!-- more -->

### 预处理打印消息

```
print a message from the C preprocessor
#if defined(__ANDROID__) && defined(__DEBUG__)
    #define OUT_LOG     1
    #warning "output log"
#else
    #define OUT_LOG     0
    #pragma message("no log")
#endif
```

<!-- more -->