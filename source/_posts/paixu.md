---
title: 常用排序算法
date: 2019/02/06
tags: 
- python
- 算法
categories: 算法
comments: 
description: 
top_img: 
---
## 排序算法的分析
### 排序算法的执行效率
1. 最好情况、最坏情况、平均情况时间复杂度
2. 时间复杂度的系数、常数、低阶
3. 比较次数和交换(或移动)次数
### 排序算法的内存消耗
原地排序，特指空间复杂度是O(1)的排序算法
### 排序算法的稳定性
有一组数据 2，9，3，4，8，3，按照大小排序之后，按照大小排序后就是 2，3，3，4，8，9；这里有两个3，如果排序后两个3的位置没有变，那么这种排序算法就是稳定的，反之就是不稳定的。
## 冒泡、插入、选择排序
此3种排序的时间复杂度皆为O(n^2)
### 插入排序
将数组中的数据分为两个区间，已排序区间和未排序区间。核心思想是取未排序区间的元素，在已排序区间找到何使的插入位置将其插入，保证已排序区间数据一致有序。重复此过程，直到未排序区间中元素为空，排序结束。
```
def insert_sort(lst):#插入排序
for i in range(1,len(lst)): # 开始片段[0,1]已排序
x = lst[i] # x取列表第二位值
j = i
while j>0 and lst[j-1] > x: # 判断未排序区间是否比排序区间的数字大
lst[j] = lst[j-1] # 反序逐个后移元素，确定插入位置
j=j-1 # j自减，从当前位置向下比较
lst[j] =x # 插入元素
print(lst) # 输出每次排序后的结果
return lst
print(insert_sort([100,99,98,111,33,3,123]))
>>
[99, 100, 98, 111, 33, 3, 123]
[98, 99, 100, 111, 33, 3, 123]
[98, 99, 100, 111, 33, 3, 123]
[33, 98, 99, 100, 111, 3, 123]
[3, 33, 98, 99, 100, 111, 123]
[3, 33, 98, 99, 100, 111, 123]
[3, 33, 98, 99, 100, 111, 123]
```
### 选择排序
选择排序与插入排序的思路类似，也分已排区间和未排序区间；但是选择排序每次会从未排序区间种找到最小的元素，将其放到已排序区间的末尾。
```
def select_sort(lst):
# 每一次内循环选出最小的值，外循环控制次数
for i in range(len(lst)-1):#总循环次数len(num)-1
k=i
for j in range(i,len(lst)):
if lst[j] < lst[k]:
k=j #选出最小的值的位置k
if i != k:
lst[i],lst[k] =lst[k],lst[i] #交换数据
print(lst)
return lst
print(select_sort([100, 99, 98, 111, 33, 3, 123]))
>>
[3, 99, 98, 111, 33, 100, 123]
[3, 33, 98, 111, 99, 100, 123]
[3, 33, 98, 111, 99, 100, 123]
[3, 33, 98, 99, 111, 100, 123]
[3, 33, 98, 99, 100, 111, 123]
[3, 33, 98, 99, 100, 111, 123]
[3, 33, 98, 99, 100, 111, 123]
```
### 冒泡排序
冒泡排序只会操作相邻的两个元素。每次冒泡排序都会对相邻的两个元素进行比较，看是否满足大小关系要求。如果不满足则互换。一次冒泡至少会让一个元素移动到它应该在的位置，重复n次，就完成了n个数据的排序工作。
```
def bubble_sort(lst):#冒泡排序
for i in range(len(lst)-1): #外循环
for j in range(1,len(lst)-i):# 内循环
if lst[j-1] >lst[j]:
lst[j-1],lst[j]=lst[j],lst[j-1]
print(lst)
return lst
print(bubble_sort([100, 99, 98, 111, 33, 3, 123]))
>>
[99, 98, 100, 33, 3, 111, 123]
[98, 99, 33, 3, 100, 111, 123]
[98, 33, 3, 99, 100, 111, 123]
[33, 3, 98, 99, 100, 111, 123]
[3, 33, 98, 99, 100, 111, 123]
[3, 33, 98, 99, 100, 111, 123]
[3, 33, 98, 99, 100, 111, 123]
```
但是上述冒泡排序是未优化的，优化思路大致如下：
1. 加入排序标志位，如果一次冒泡操作没有数据交换时，说明已经达到完全有序，不用再进行后续的冒泡操作，退出此次冒泡即可；
2. 将列表进行左右同时冒泡排序，减少循环次数
```
def bubble_sort(lst):
# 最优冒泡算法，左右双向排序，同时进行
for j in range(len(lst)):
flag = False # 结束标志位
for i in range(j+1,len(lst)-j):
if lst[i-1]>lst[i]: #正向选择最大，前一位比后一位大
lst[i-1],lst[i] = lst[i],lst[i-1]
flag = True
if lst[len(lst)-i-1]>lst[len(lst)-i]: #反向选择最小，前一位比后一位小
lst[len(lst) - i - 1],lst[len(lst)-i]=lst[len(lst)-i],lst[len(lst)-i-1]
flag = True
else:
flag =True
pass
if not flag:
break
print(lst)
return lst
print(bubble_sort([100, 99, 98, 111, 33, 3, 123]))
>>
[3, 99, 98, 100, 33, 111, 123]
[3, 33, 98, 99, 100, 111, 123]
[3, 33, 98, 99, 100, 111, 123]
[3, 33, 98, 99, 100, 111, 123]
```
由上述结果可看出，优化后的冒泡在第二次排序后就已经排序好了。
## 快排，归并排序
快速排序和归并排序的时间复杂度都是O(nlogn)
### 归并排序
归并排序的核心思想是，如果要排序一个数组，先把数组从中间分成两个部分，然后对两个部分分别排序，再将排序好的结果合并到一起，这样整个数组就有序了。
```
def merge(left, right):
i, j = 0, 0
result = []
while i < len(left) and j < len(right): # 比较传入的两个子序列，对两个子序列进行排序
if left[i] <= right[j]:
result.append(left[i])
i += 1
else:
result.append(right[j])
j += 1
result.extend(left[i:]) # 将排好序的子序列合并
result.extend(right[j:])
print(result)
return result
def merge_sort(lst):
if len(lst) <= 1:
return lst # 从递归中返回长度为1的序列
middle = int(len(lst) / 2)
left = merge_sort(lst[:middle]) # 通过不断递归，将原始序列拆分成n个小序列
right = merge_sort(lst[middle:])
return merge(left, right)
print(merge_sort([100, 99, 98, 111, 33, 3,123]))
>>
[98, 99]
[98, 99, 100]
[33, 111]
[3, 123]
[3, 33, 111, 123]
[3, 33, 98, 99, 100, 111, 123]
[3, 33, 98, 99, 100, 111, 123]
```
### 快速排序
快速排序的思想是如果要排序数组中下标从p到r之间的一组数据，我们选择p到r之间的任意一个数据作为pivot(分区点)
然后遍历p到r之间的数据，将小于pivot的放到左边，大于pivot的放到右边，pivot则在中间；这样这个数组则被分为三个部分p—>pivot-1,pivot,pivot+1->r，然后根据递归的思路，不断的递归排序下标p->pivot-1，pivot+1->r,直到区间缩为1.
```
def partition(lst, p, r): #原地分区函数
i = p # 定义游标i，将lst分为两个区间，与选择排序类似
for j in range(p, r):
if lst[j] <= lst[r]:
lst[i], lst[j] = lst[j], lst[i]
i += 1
lst[i], lst[r] = lst[r], lst[i]
print(lst)
return i
def quicksort(lst, p, r):
if p < r:
pivot = partition(lst, p, r)
quicksort(lst, p, pivot - 1)
quicksort(lst, pivot + 1, r)
return lst
lst = [100, 99, 98, 111, 33, 3, 123]
print(quicksort(lst, 0, len(lst)-1))
>>
[100, 99, 98, 111, 33, 3, 123]
[3, 99, 98, 111, 33, 100, 123]
[3, 99, 98, 33, 100, 111, 123]
[3, 33, 98, 99, 100, 111, 123]
[3, 33, 98, 99, 100, 111, 123]
[3, 33, 98, 99, 100, 111, 123]
```

