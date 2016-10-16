---
date: 2015-11-15 
title: Java-归并排序和快速排序的效率比较
categories: 算法
tags: [java]
---


用java完整地实现了归并排序和快速排序，然后测试两个算法在数组长度由100增加到一亿过程中的时间复杂度，比较两个算法效率。

## 归并排序

``` java
public class MergeSort {

	public int[] A;

	public MergeSort(int[] array) {
		this.A = array.clone();
		sort(0, array.length - 1);
	}

	public void sort(int low, int high) {

		if (low < high) {
			int mid = (low + high) / 2;
			sort(low, mid);
			sort(mid + 1, high);
			merge(low, mid, high);
		}
	}

	public void merge(int low, int mid, int high) {

		// 声明新的数组，临时储存归并结果
		int[] B = new int[high - low + 1];
		int h = low;
		int i = 0;
		int j = mid + 1;

		while (h <= mid && j <= high) {
			if (A[h] <= A[j]) {
				B[i] = A[h];
				h++;
			} else {
				B[i] = A[j];
				j++;
			}
			i++;
		}

		// 等号很重要
		if (h <= mid) {
			for (int k = h; k <= mid; k++) {
				B[i] = A[k];
				i++;
			}
		} else {
			for (int k = j; k <= high; k++) {
				B[i] = A[k];
				i++;
			}
		}

		for (int k = low; k < high; k++) {
			A[k] = B[k - low];
		}

	}
}
```

## 快速排序

``` java
public class QuickSort {

	public int[] A;

	public QuickSort(int[] array) {
		this.A = array.clone();
		sort(0, array.length - 2);
	}

	public void sort(int p, int q) {
		if (p < q) {
			int j = partition(p, q);
			sort(p, j - 1);
			sort(j + 1, q);
		}
	}

	public int partition(int p, int q) {
		int j = q;
		int axis = A[p];
		int i = p;

		while (true) {

			//从左往右找到第一个比axis大的数字
			while (A[i] <= axis) {
				i++;
			};

			//从右往左找到第一个小于等于axis的数字
			while (A[j] > axis) {
				j--;
			};

			if (i < j) {
				swap(i, j);
			} else {
				break;
			}
		}

		A[p] = A[j];
		A[j] = axis;

		return j;
	}

	public void swap(int i, int j) {
		int tmp = A[i];
		A[i] = A[j];
		A[j] = tmp;
	}

}
```

## 效率比较

新建TimeCounter类，用来实例化两个算法类，并测试两个算法。size指定测试数组的长度，maximum 指定生成的数组中元素的最大值，注意：由于快排的数组最后一个数字要求最大，因此实际的数组长度是是size+1

```
import java.util.Arrays;
import java.util.Random;

public class TimeCounter {

	public static void main(String[] args) {
		// TODO Auto-generated method stub

		int size = 101;
		int maximum = 100000;
		int[] array = new int[size];

		for (int i = 0; i < array.length - 1; i++) {
			array[i] = new Random().nextInt(maximum);
		}

		//快排数组中的最后一个数字最大
		array[size - 1] = maximum + 1;
 
		long QuickStart = System.currentTimeMillis();
		new QuickSort(array);
		long QuickEnd = System.currentTimeMillis();

		long MergeStart = System.currentTimeMillis();
		MergeSort mergesort=new MergeSort(array);
		long MergeEnd = System.currentTimeMillis();

//		 System.out.println(Arrays.toString(array));
//		 System.out.println(Arrays.toString(mergesort.A));

		System.out.println("quick sort: " + (QuickEnd - QuickStart));
		System.out.println("merge sort: " + (MergeEnd - MergeStart));

	}
}

```

在同一台计算机上，修改size的值，运行TimeCounter.java，得到两个算法在不同数组长度下的执行时间：

| 数组长度| 快速排序（运行时间/毫秒） | 归并排序（运行时间/毫秒） |
| ------------- |:-------------:| :-----:|
| 100 | 0 | 0 |
| 1000 | 1 | 1 |
| 10000 | 1 | 3 |
| 100000 | 14 | 14 |
| 1000000 | 79 | 120 |
| 10000000 |982 | 1186 |
| 100000000	| 55733| 12328	 |


## 结果

我们知道快排和归并的理论上的时间复杂性如下表：

| 算法 | 最坏时间复杂性 | 平均时间复杂性 
| ------------- |:-------------:|:-------------:|
| 快速排序 | n^2 | n*log(n) |
| 归并排序 | n*log(n)  |  n*log(n) |


**1. 在数组长度小于一千万的时候，如下图，快速排序的速度要略微快于归并排序，可能是因为归并需要额外的数组开销（比如声明临时local数组用来储存排序结果），这些操作让归并算法在小规模数据的并不占优势。**

![这里写图片描述](http://img.blog.csdn.net/20161016162825792)

**2. 但是，当数据量达到亿级时，归并的速度开始超过快速排序了，如下图，因为归并排序比快排要稳定，所以在数据量大的时候，快排容易达到O(n^2)的时间复杂度，当然这里是指未改进的快排算法。**

![这里写图片描述](http://img.blog.csdn.net/20161016162835527)
