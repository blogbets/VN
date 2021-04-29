---
layout: post
title: c语言专题及参考分析
date:   2014-11-10 22:06:05
categories: C
tags: [c语言专题]
---

* content
{:toc}

本文以专题形式列出c语言中几个重难点，专题后新附加了参考分析，如有疑问请留言。
---
<!--more-->

##专题一 结构占用内存长度

在linux/windows上运行下面一段程序，你能总结出struct内存对齐规则吗？
```
struct ta {
	short b;
        long a;
        char c;
};

struct tb {
	short b;
        int a;
        char c;
};

struct tc {
        short a;
        char  b;
        short c;
        };

printf("size ta:%d tb:%d tc:%d\n", sizeof(struct ta), sizeof(struct tb), sizeof(struct tc));
```
参考分析：struct内存总是以其L最长内存占字节成员a为基准来对齐的，如果a后面相邻成员所占内存字节和小于或等于nL(n为正整数)则内存拼接,
其中sizeof(long)结果取决于操作系统和cpu结构，各操作系统和cpu架构下sizeof(long)如下表：
 OS           arch           size
Windows       IA-32        4 bytes
Windows       Intel 64     4 bytes
Windows       IA-64        4 bytes
Linux         IA-32        4 bytes
Linux         Intel 64     8 bytes
Linux         IA-64        8 bytes
Mac OS X      IA-32        4 bytes
Mac OS X      Intel 64     8 bytes  
(详见：https://software.intel.com/en-us/articles/size-of-long-integer-type-on-different-architecture-and-os)

假设改程序运行于Linux IA-64上则结果为：
size ta:24 tb:12 tc:6

##专题二 指针和数组
运行下面程序，观察数组array地址&array输出结果，另外观察程序中指针的声明，你能学到什么？

```
#include <stdio.h>

int g_array[] = {2,3,4,5,6,7,8,9, 10};

int func1(int a)
{
	printf("%s be called\n", __func__);
	return a;
}

int func2(int a)
{
	printf("%s be called\n", __func__);
	return a;
}

int (*func3(int array[4]))[9]
{
	int (*r)[9] = &g_array;
	return r;
}

char *const *func4()
{
        char *const str = "abc";
        char *const *p = &str;
        return p;
}

void addr1(int array[]) 
{
	/*此处array被编译器转化为指针，array和&array不再相等,结果为&array[1] - 4 == &array[0] == array != &array*/
	printf("%x %x %x %x\n", array, &array, &array[0], &array[1]);
}

void addr2(int *array) 
{
	/*此行代码在gcc和llvm上的结果不同*/
	printf("%x %x %x %x %x\n", array, &array, &array[0], &array[1], ++array);
}

int main()
{
	
	int array[4] = {1, 2, 3, 4};
	int( *p)[4] = &array;
	int (*pfunc[])(int) = {func1, func2};
	printf("%d, %d, %d\n", (*p)[1], pfunc[1](3), (*func3(array))[1]);
	
	addr1(array);
	addr2(array);
	printf("%#x %#x %#x %#x\n", array, &array, &array[0], &array[1]);
	addr1(g_array);
	addr2(g_array);

	printf("%#x %#x %#x %#x\n", g_array, &g_array, &g_array[0], &g_array[1]);	
	char * const *(*next)() = func4;
	
	return 0;
}
```
参考分析：
gcc编译器输出：
func2 be called
2, 3, 3
bfe69a68 bfe69a40 bfe69a68 bfe69a6c
bfe69a6c bfe69a40 bfe69a6c bfe69a70 bfe69a6c
0xbfe69a68 0xbfe69a68 0xbfe69a68 0xbfe69a6c
80498c0 bfe69a40 80498c0 80498c4
80498c4 bfe69a40 80498c4 80498c8 80498c4
0x80498c0 0x80498c0 0x80498c0 0x80498c4

llvm编译器输出：
func2 be called
2, 3, 3
5e72ebf0 5e72eb88 5e72ebf0 5e72ebf4
5e72ebf0 5e72eb88 5e72ebf0 5e72ebf4 5e72ebf4
0x5e72ebf0 0x5e72ebf0 0x5e72ebf0 0x5e72ebf4
14d2040 5e72eb88 14d2040 14d2044
14d2040 5e72eb88 14d2040 14d2044 14d2044
0x14d2040 0x14d2040 0x14d2040 0x14d2044

gcc、llvm对printf("%x %x %x %x %x\n", array, &array, &array[0], &array[1], ++array);这行处理不同，
你观察到了吧，另外还有四个有趣的问题：
<br>1. addr1、addr2中array！= &array,main函数中array == &array。
（数组作为函数参数在函数内被转化为指向数组首地址的指针，对指针取地址就是另外的值了）
<br>2. gcc运行这行（++array后)，&array却没有变。
<br>3. func2 be called却最先打印。
<br>4. 函数中涉及到的指针

```
int (*pfunc[])(int)；//指向函数数组的指针
int (*func3(int array[4]))[9]；//函数指针，其形参为int [4], 返回长度为9的int指针向量（等价于int [][9]）
int( *p)[4];//长度为4的int指针向量,可以这么使用，
int a[][4] = {1, 2, 3, 4, 5, 6, 7, 8};
int (*p)[4] = a;
printf("%d, %d, %d\n", p[0][1], (*p)[1], (*(p + 1))[1]);//结果：2, 2, 6
```

##专题三 switch - case容易忽视的细节
下面这段程序结果是什么？
```
int main()
{
        int i = 1, a = 0;
        switch (i) {
                a = 1;
                case 0:
                        printf("0\n");
                break;
                case 1:
                        printf("1\n");
                default:
                        break;
        }

        printf("a:%d\n", a);
        return 0;
}
```
参考分析：
编译器忽略switch和case之间的语句，故答案为：0

##专题四 typedef用法要诀
初识typedef你是否不知所云，那么请往下看。

语法：typedef 变量声明
变量名字即代表所声明的类型，例如，typedef char T[2], T t[3]为一个3行2列的二维数组,
又如，
```
typedef struct student{
	int id;
	int age;
	char sex;
	char name[32];
} student_t;
``` 
student_t zhangsan;等价于 struct student{...} zhangsan;      

