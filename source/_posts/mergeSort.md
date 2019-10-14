---
title: 归并排序
date: 2018-04-10 15:11:15
tags:
  - Arithmetic
---
归并排序

## 核心思想

  归并排序是分治法的运用、所谓分治法就是将一个大问题分成一个个小的具有相同性质的问题。在归并排序中对于一个无序数组我们可以将数组先从中间分成两个数组然后对这两个数组进行排序。排序好了之后再将这两个有序的数组合并在一起。这个时候对于两个数组的排序就可以复用我们刚才的思想。

### 分的思想  

![](/img/mergeSort-1.png)

### 治的思想  

![](/img/mergeSort-2.png)
![](/img/mergeSort-3.png)

## 算法实现

```Java
public class MergeSort {
    public static void main(String []args){
        int []arr = {9,8,7,6,5,4,3,2,1};
        sort(arr);
        System.out.println(Arrays.toString(arr));
    }
    public static void sort(int []arr){
        int []temp = new int[arr.length];//在排序前，先建好一个长度等于原数组长度的临时数组，避免递归中频繁开辟空间
        sort(arr,0,arr.length-1,temp);
    }
    private static void sort(int[] arr,int left,int right,int []temp){
        if(left<right){
            int mid = (left+right)/2;
            sort(arr,left,mid,temp);//左边归并排序，使得左子序列有序
            sort(arr,mid+1,right,temp);//右边归并排序，使得右子序列有序
            merge(arr,left,mid,right,temp);//将两个有序子数组合并操作
        }
    }
    private static void merge(int[] arr,int left,int mid,int right,int[] temp){
        int i = left;//左序列指针
        int j = mid+1;//右序列指针
        int t = 0;//临时数组指针
        while (i<=mid && j<=right){
            if(arr[i]<=arr[j]){
                temp[t++] = arr[i++];
            }else {
                temp[t++] = arr[j++];
            }
        }
        while(i<=mid){//将左边剩余元素填充进temp中
            temp[t++] = arr[i++];
        }
        while(j<=right){//将右序列剩余元素填充进temp中
            temp[t++] = arr[j++];
        }
        t = 0;
        //将temp中的元素全部拷贝到原数组中
        while(left <= right){
            arr[left++] = temp[t++];
        }
    }
}
```
## 总结
  归并排序是一种稳定的排序、时间复杂度为O(nlogn)。在Java中我们常用Arrays.sort()内部使用的是TimeSort的排序方式。TimeSort是对归并排序的一种优化。
## 参考
http://bubkoo.com/2014/01/15/sort-algorithm/merge-sort/

https://www.cnblogs.com/chengxiao/p/6194356.html
