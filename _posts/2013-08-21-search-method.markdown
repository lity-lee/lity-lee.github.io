---
layout: post
category: program
title:  "查找算法(c++实现)"
tags: [查找,c++,编程]
---

<!-- more -->

### 线性查找

```
int data[100];
int search;
int location = -1;
for(int i=0, i<data.length; i++)
{
  if(search == data[i])
  {
    location = i;
    break;
  }else {
    // don't handle
  }
}

```

### 二分查找(针对有序数组)

```
int data[100];
int search;
int location = -1;
int min=0, max = data.length -1;
while(min <= max)
{
  int mid = (min + max)/2;
  if(search == data[mid])
  {
    location = i;
    break;
  }else if(search > data[mid])
  {
    min = mid + 1;
  }else if(search < data[mid])
  {
    max = mid-1;
  }else {
    // error case
  }
}
```
