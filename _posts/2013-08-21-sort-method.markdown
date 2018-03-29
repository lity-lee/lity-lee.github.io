---
layout: post
category: program
title:  "排序算法(c++实现)"
tags: [排序,c++,编程]
---

<!-- more -->

### 冒泡排序

```
int data[100];
for(int i=0; i<data.length; i++)
{
	for(int j=0; j < data.length-1-i; j++)
	{
		if(data[j] > data[j+1])
		{
			int temp = data[j];
			data[j] = data[j+1];
			data[j+1] = temp;
		}else {
			// don't handle
		}
	}
}
```

### 选择排序

```
int data[100];
for(int i=0; i<data.length; i++)
{
	int min = i;
	for(int j=i+1; j<data.length; j++)
	{
		if(data[j] < min)
		{
			min = i;
		}else
		{
			// don't handle
		}
		int temp = data[i];
		data[i] = data[j];
		data[j] = temp;
	}
}
```

### 插入排序

```
int data[100];
for(int i=1; i<data.length; i++)
{
	int insert = data[i];
	int moveItem = i;
	while(data[moveItem > 0 && data[moveItem-1] > insert)
	{
		data[moveItem] = data[moveItem-1];
		moveItem--;
	}
	data[moveItem] = insert;
}
```
