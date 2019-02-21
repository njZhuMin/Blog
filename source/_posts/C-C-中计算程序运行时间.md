---
title: C/C++中计算程序运行时间
date: 2016-03-26 01:42:19
tags: [C,CPP]
categories:
- data-structure
- misc
---

# 最常用的的方式：

`clock()`计算的是CPU执行耗时，注意是CPU！如果有多个核并行，最后的结果是每个CPU上运算时间的总和！想要精确到毫秒，可以使用：

一般来说，只要求精确到秒的话，`time()`是很好使的
```cpp
#include <stdio.h>
#include <time.h>
int main(){
    time_t t_start, t_end;
    t_start = time(NULL) ;
    sleep(3000);
    t_end = time(NULL) ;
    printf("time: %.0f s \n", difftime(t_end, t_start)) ;
    return 0;
}
```

<!-- more -->

如果要让程序休眠3秒，Windows使用`Sleep(3000)`，Linux使用`sleep(3)`，即Windows的Sleep接口的参数的单位是毫秒，Linux的sleep接口的参数的单位是秒。
如果需要精确到毫秒，以上程序就发挥不了作用，如果在Java要达到这要求就很简单了，代码如下所示：
```java
public class Time {
    public static void main(String[] args) {
        try {
            long startTime = System.currentTimeMillis();
            Thread.sleep(3000);
            long endTime = System.currentTimeMillis();
            System.out.println("time: " + (endTime - startTime) + " ms");
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```
通过Google找了一些资料后，发现C语言里没有标准的接口可以获得精确到毫秒的时间，都会调用到与操作系统相关的API，下面会分别介绍在Linux和Windows系统下的多种实现方法。

# Linux系统

## 使用`gettimeofday()`接口：

```cpp
#include <stdio.h>
#include <sys/time.h>
int main() {
    struct timeval start, end;
    gettimeofday(&start, NULL);
    sleep(3);
    gettimeofday(&end, NULL);
    int timeuse = 1000000 * (end.tv_sec - start.tv_sec) + end.tv_usec -start.tv_usec;
    printf("time: %d us \n", timeuse);
    return 0;
}
```
`gettimeofday()`能得到微秒数，比毫秒还要更精确。

## 使用`ftime()`接口：

```cpp
#include <stdio.h>
#include <sys/timeb.h>
long long getSystemTime() {
    struct timeb t;
    ftime(&t);
    return 1000 * t.time + t.millitm;
}

int main() {
    long long start=getSystemTime();
    sleep(3);
    long long end=getSystemTime();
    printf("time: %lld ms \n", end-start);
    return 0;
}
```

# Windows系统

## 使用`GetTickCount()`接口：

```cpp
#include <</span>windows.h>
#include <</span>stdio.h>
int main() {
    DWORD start, stop;
    start = GetTickCount();
    Sleep(3000);
    stop = GetTickCount();
    printf("time: %lld ms \n", stop - start);
    return 0;
}
```
Windows系统下有些编译器使用`printf`输出64位整数参数要使用`%I64d`，比如VC。

## 使用`QueryPerformanceX()`接口：

```cpp
#include <windows.h>
#include <stdio.h>
int main(){
    LARGE_INTEGER li;
    LONGLONG start, end, freq;
    QueryPerformanceFrequency(&li);
    freq = li.QuadPart;
    QueryPerformanceCounter(&li);
    start = li.QuadPart;
    Sleep(3000);
    QueryPerformanceCounter(&li);
    end = li.QuadPart;
    int useTime =(int)((end - start) * 1000 / freq);
    printf("time: %d ms \n", useTime);
    return 0;
}
```

## 使用`GetSystemTime()`接口：

```cpp
#include <</span>windows.h>
#include <</span>stdio.h>
int main(){
     SYSTEMTIME currentTime;
     GetSystemTime(&currentTime);
     printf("time: %u/%u/%u %u:%u:%u:%u %d \n", 
     currentTime.wYear,currentTime.wMonth,currentTime.wDay,
     currentTime.wHour,currentTime.wMinute,currentTime.wSecond,
     currentTime.wMilliseconds,currentTime.wDayOfWeek);
     return 0;
}
```
这种方法没给出计算时间差的实现，只给出如何用`GetSystemTime()`调用得到当前时间，计算时间差比较简单，根据年、月、日、时、分秒和毫秒计算出一个整数，再将两整数相减即可。
