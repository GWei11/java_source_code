# ForkJoin 的作用
* 适用于大的任务比较耗时，而且任务可以拆分的情形，原来的大任务执行就一个线程，现在任务拆分后是多个线程去执行。
* 任务拆分的逻辑需要自己实现。

# ForkJoin的使用步骤
## 代码示例
```java
package com.example.java8;

import java.util.Arrays;
import java.util.List;
import java.util.concurrent.ForkJoinPool;
import java.util.concurrent.RecursiveTask;
public class ForkJoinDemo {
    public static void main(String[] args) {
        int[] tempData = new int[1000];
        for (int i = 0; i < 1000; i++) {
            tempData[i] = i;
        }
        ForkDemo forkDemo = new ForkDemo(0, 1000, tempData);
        Integer result = ForkJoinPool.commonPool().invoke(forkDemo);
        System.out.println(result);
    }
}

class ForkDemo extends RecursiveTask<Integer> {
    private int start;
    private int end;
    private int[] data;
    private final int THRESHOLD = 250;

    public ForkDemo(int start, int end, int[] data) {
        this.start = start;
        this.end = end;
        this.data = data;
    }

    @Override
    protected Integer compute() {
        int sum = 0;
        // 任务足够小，就不需要使用子任务（既然是任务拆分，肯定就需要拆分的条件，这里就是拆分的条件）
        if (this.end - this.start <= THRESHOLD) {
            for (int i = this.start; i < this.end; i++) {
                sum += data[i];
            }
        } else {
            int middle = (this.start + this.end) / 2;
            System.out.println(String.format("split %d~%d ==> %d~%d, %d~%d", start, end, start, middle, middle, end));
            ForkDemo demoOne = new ForkDemo(this.start, middle, data);
            ForkDemo demoTwo = new ForkDemo(middle, this.end, data);
            invokeAll(demoOne, demoTwo);
            Integer resultOne = demoOne.join();
            Integer resultTwo = demoTwo.join();
            sum = resultOne + resultTwo;
            System.out.println("result = " + resultOne + " + " + resultTwo + " ==> " + sum);
            System.out.printf("start: %d, middle:%d, end:%d \n", start, middle, end);
        }
        return sum;
    }
}
```

## 执行结果
```java
split 0~1000 ==> 0~500, 500~1000
split 0~500 ==> 0~250, 250~500
split 500~1000 ==> 500~750, 750~1000
result = 31125 + 93625 ==> 124750
start: 0, middle:250, end:500 
result = 156125 + 218625 ==> 374750
start: 500, middle:750, end:1000 
result = 124750 + 374750 ==> 499500
start: 0, middle:500, end:1000 
499500
```
* 该案例是计算 1-1000 的和，使用 ForkJoin 将其拆分成了四个小任务，每一个任务计算其中的一段，最后将每一个任务的结果合起来
就是最终的结果。
* 对于任务是如何进行拆分的逻辑是需要自己实现的。