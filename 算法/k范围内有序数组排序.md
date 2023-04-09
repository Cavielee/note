# 描述

数组中的每一个数，只需移动k范围内即可变为有序数组时所在的位置。

```java
/**
 * 一个几乎有序的数组，其排完序后，每一个数字移动的距离不会超过k。（k值不会很大）
 * 选择一个排序算法排序（堆排序）
 * @author CavieLee
 * @since 2022/03/23
 */
public class SortKRange {
    public static int[] arr = new int[]{3, 0, 5, 9, 4, 7};

    public static void sortedArrDistanceLessK(int[] arr, int k) {
        // 其内部实现就是小根堆
        PriorityQueue<Integer> heap = new PriorityQueue<>();
        int index = 0;
        // 将前k个数加入到小根堆，此时小根堆为有序
        for (; index < Math.min(arr.length, k); index ++) {
            heap.add(arr[index]);
        }
        int i = 0;
        // 每添加一个数，就弹出一个数。
        // 由于题目确保，因此弹出的数一定是[i,i+k]范围内最小的数（小根堆弹出即为最小）
        for (; index < arr.length; i++, index++) {
            heap.add(arr[index]);
            arr[i] = heap.poll();
        }
        // 剩下的直接弹出即为有序的数
        while (!heap.isEmpty()) {
            arr[i++] = heap.poll();
        }
    }

    public static void main(String[] args) {
        sortedArrDistanceLessK(arr, 2);
    }
}

```

