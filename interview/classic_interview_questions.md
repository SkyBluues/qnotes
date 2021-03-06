# 面试题目汇集

## 多路归并排序（muduo-12.8）

### 问题描述

计算机内存4G，磁盘上有一个100G大小的文件，保存了某种类型的数据若干个。对这个文件排序，将排序结果输出到磁盘上。以升序为例。

### 解题思路

1. 将大文件分割成多个小文件，如题中100G的文件可分割为100个1G大小的文件。
2. 分别对分割后的文件排序（1G可以读入内存），将排序后的结果输出到磁盘上。
3. 对排序后的100个文件进行多路归并排序。

### 多路归并排序的思路

使用一个大小为100的小堆，按顺序同时读取100个文件，每读入一个元素，就将元素及其对应的文件句柄组成一个pair，push进堆。当堆中的元素达到100个时，堆顶元素即为最小的元素，将其pop出堆并写入磁盘。然后继续读入被pop出去的元素对应的文件的下一个元素，push进堆，重复上述过程，直到所有文件读完。

### 优化思路

分割大文件，对100个小文件进行排序，可以考虑并行处理。比如第一个文件在排序时，第二个文件可以读入内存，让计算和IO同时进行。

由于磁盘顺序读写比随机读写快。若读和写都对应同一块磁盘，不考虑两个线程同时读写磁盘（因为多线程同时读写，在操作系统层面会排队，导致两个线程之间切换，随机访问）。若读和写对应不同的两个磁盘，可以考虑读写各一个线程，进一步提高并行度。

### 同类题目

问题：计算机内存4G，有两个100G的文本文件，每个文件单行最多1K。两个文件有少量重复的行，找出重复的行并输出。
思路：文件a顺序读入内存，按行求哈希并取模，此处模100，将文件分成100个小文件a0, a1, a2, a3, a4, a5,...，文件b同样处理。然后对a0，b0求交集，a1，b1求交集...
思路2：文件a分割成若干排序的小文件，文件b分割成若干排序的小文件，对a,b分别进行多路归并排序，输出的同时对a,b求交集。

问题：有几个大文本文件，将所有文件按每行出现的次数排序。
思路：每个文件按照行进行哈希取模，分成多个小文件，每个小文件内记录每条记录出现的次数；多个文件的小文件进行合并；最后用小文件进行多路归并（以词条出现的次数为key）

## std::cout 原理

### 问题描述

1. std::cout 从调用到打印出来，经历哪些函数，哪些系统调用，输出到哪个文件
2. std::cout 如果每个输出一个字符，会即使打印吗？什么时候会刷缓存？

## 文件系统

### 问题描述

文件系统中，一个文件夹下有很多小文件，对小文件的访问会变慢，原因是什么？

## 代码从文本到在操作系统运行起来的过程

### 问题描述

描述代码从一个个文本文件，到在操作系统运行起来的完整过程。包括编译、链接、装载等过程。

## 进程在内存中的布局

### 问题描述

二进制程序装载到内存以后，在内存虚拟地址空间的布局。

## 线程与进程

### 问题描述

说说进程与线程的区别和联系。

## 静态库、动态库、静态链接、动态链接

### 问题描述

描述静态链接与动态链接的区别、优缺点。

## malloc/free/new/delete

### 问题描述

1. malloc/free和new/delete的区别联系
2. new/delete的原理
3. 用new申请一个数组，delete不加[]会如何
4. 用new申请一个数组，free会如何

### 参考
1. https://blog.csdn.net/nodeathphoenix/article/details/39693865#:~:text=%E8%B0%83%E7%94%A8%E6%9E%90%E6%9E%84%E5%87%BD%E6%95%B0%E7%9A%84%E6%AC%A1%E6%95%B0%E6%98%AF%E4%BB%8E%E6%95%B0%E7%BB%84%E5%AF%B9%E8%B1%A1%E6%8C%87%E9%92%88%E5%89%8D%E9%9D%A2%E7%9A%84%204%20%E4%B8%AA%E5%AD%97%E8%8A%82%E4%B8%AD%E5%8F%96%E5%87%BA%EF%BC%9B%20%E4%BC%A0%E5%85%A5%20operator%20delete,%5B%5D%20%E5%87%BD%E6%95%B0%E7%9A%84%E5%8F%82%E6%95%B0%E4%B8%8D%E6%98%AF%E6%95%B0%E7%BB%84%E5%AF%B9%E8%B1%A1%E7%9A%84%E6%8C%87%E9%92%88%20pAa%EF%BC%8C%E8%80%8C%E6%98%AF%20pAa%20%E7%9A%84%E5%80%BC%E5%87%8F%204%E3%80%82%20%E4%B8%8A%E8%BF%B0%E5%9B%BE%E7%89%87%E4%B8%AD%E7%BA%A2%E9%A2%9C%E8%89%B2%E7%9A%84%E6%95%B0%E5%AD%973%E5%8D%B3%E5%AD%98%E5%82%A8%E5%9C%A8%E5%9B%9B%E4%B8%AA%E5%AD%97%E8%8A%82%E4%B8%AD%E7%9A%84%E6%95%B0%E7%BB%84%E9%95%BF%E5%BA%A6%E3%80%82
2. https://www.cnblogs.com/ywliao/articles/8116622.html

## 孤儿进程、僵尸进程

### 问题描述

什么是孤儿进程？什么是僵尸进程？

## python多线程

### 问题描述

python多线程为什么慢？
