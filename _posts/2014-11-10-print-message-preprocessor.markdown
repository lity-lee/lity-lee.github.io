---
layout: post
category: linux
title:  "print a message from the C preprocessor"
tags: [gcc,编译,编程]
---

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

