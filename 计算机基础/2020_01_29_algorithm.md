# Algorithm

## 1. Fundamentals

- ### 1.1 Basic Programming Model
  
  - c++的char为8位，java的char为16位
  - 运算优先级：!, &&, ||, */%, +-，尽量用括号
  - 数组初始化过程
  
    ```java
    // long form
    double[] a;  // declaration
    a = new double[N];  // creation
    for (int i=0; i<N; ++i) { a[i]=0.0; }  // initialization

    // short form
    double[] a = new double[N];

    // initializing declaration
    int[] a = {1,2,3,4,5};
    ```

  - 数组浅拷贝

    ```java
    int[] a = new int[N];
    int[] b = a;  // b和a指向同一内存
    ```

  - 二维数组

    ```java
    double[][] a = new double[M][N];
    ```

  - java在需要时自动将任意内置类型转为string，用户也可以主动使用toString函数

- ### 1.2 Data Abstraction

  - 对每个用户类，java都自动加上`toString()`、`equals()`、`compareTo()`、`hashCode()`等方法，用户不重写则根据该类的内存地址来确定返回值

  - java中非基本类的拷贝构造和非基本类的赋值都是类的引用的复制，指向同个内存地址（而c++中则由拷贝构造函数、赋值函数进行定义）
  
  - java中的参数传递对非基本类来说是通过reference传递的（而基本类型则是pass-by-value，含String）；return时也是如此
  
  - array数组在参数传递时使用reference，在等号右边时也是传的reference（与c++中数组的行为一致）

  - `final`修饰的变量只能被赋值一次，无论是初始化赋值还是一般赋值

- ### 1.3 Bags, Queues, and Stacks

  - Bags：只进不出，乱序

  - array、list 实现 bags、queues、stacks

- ### 1.4 Analysis of Algorithms

  - 算法的时间复杂度和空间复杂度

- ### 1.5 Case Study: Union-Find

  - 并查集

## 2. Sorting

- ### 2.1 Elementary Sorts
  
  - 选择排序

  - 插入排序

  - 希尔排序

- ### 2.2 Merge Sort

  - top-down 归并

  - bottom-up 归并

- ### 2.3 Quick Sort

  - 快排

  - 3路快排（"小于"、"等于"、"大于"）

- ### 2.4 Priority Queues

  - 堆

  - 堆排序

- ### 2.5 Applications

## 3. Searching

- ### 3.1 Symbol Tables

  - 使用非排序链表实现

  - 使用排序数组+二分查找实现

- ### 3.2 Binary Search Trees

  - 二叉搜索树

- ### 3.3 Balanced Search Trees

  - 2-3-搜索树（2-3 B树）

  - 红黑树

- ### 3.4 Hash Tables

  - 哈希表

- ### 3.5 Applications

## 4. Graphs

- ### 4.1 Undirected Graphs

  - 邻接表、邻接矩阵

  - 深度优先遍历

  - 宽度优先遍历

- ### 4.2 Directed Graphs

  - 拓扑排序

- ### 4.3 Minimum Spanning Trees

  - 最小支撑树

- ### 4.4 Shortest Paths

  - 最短路径

## 5. Strings

- ### 5.1 String Sorts

- ### 5.2 Tries

- ### 5.3 Substring Search

- ### 5.4 Regular Expressions

- ### 5.5 Data Compression

## 6. Context