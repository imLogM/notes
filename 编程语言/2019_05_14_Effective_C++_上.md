# Effective C++ （上）

作者：LogM

本文原载于 [https://segmentfault.com/u/logm/articles](https://segmentfault.com/u/logm/articles)，不允许转载~

## 1. 让自己习惯C++

- ### 1.1 条款01：把C++看做多种语言的集合

  - #### 1.1.1 C++的发展：

    - 阶段1：C with Classes；
    - 阶段2：加入异常（exceptions）、模板（templates）、STL库；
    - 阶段3：过程形式 + 面向对象形式 + [函数形式](https://alexiachen.github.io/blog/2017/03/03/functional-programming-cpp11/) + [泛型形式](https://www.jianshu.com/p/1452033e0629) + [元编程形式](https://www.jianshu.com/p/b56d59f77d53)

  - #### 1.1.2 如何使用：视为四个次语言的组合

    - C。最基础的C。
    - 面向对象的C。C with Classes。
    - Template C++。泛型编程，更深奥的还有模板元编程（TMP），但一般人用不到。
    - STL。作者认为STL有自己的一套规约，可以单独拎出来。

- ### 1.2 条款02：尽量以const, enum, inline替换 #define

  - #### 1.2.1 #define用于常量的场景

    - 缺点：
      - a. #define定义的常量导致编译出错时，出错提示不友好；
      - b. #define无视作用域（scope），无封装性。
    - 解决：
      - a. 大多数情况可以用const替换；
      - b. 旧编译器对类内static const语法较苛刻，此时可用enum替换#define。

  - #### 1.2.2 #define用于宏（函数）的场景

    - 缺点:
      - a. 需要注意括号的使用，给自己添麻烦；
      - b. #define无视作用域（scope），无封装性。
    - 解决：使用inline函数。

- ### 1.3 条款03：尽可能使用const

  - 原因：让编译器辅助对常量/常量函数的错误使用。

  - #### 1.3.1 const常量

    - ```cpp
      const char* p = ...;  // p可改变，*p不可改变
      char const * p = ...;  // 同上，有些人会这么写
      char* const p = ...;  // p不可改变，*p可改变
      const char* const p = ...;  // p不可改变，*p不可改变

      const std::vector<int>::iterator iter = ...;  // iter不可改变，*iter可改变
      std::vector<int>::const_iterator iter = ...;  // iter可改变，*iter不可改变

      const Rational operator* (const Rational& lhs, const Rational& rhs); // 用于避免出现 (a * b) = c 这样的代码
      ```

  - #### 1.3.2 const函数

    - ```cpp
      bool func(int a, int b) const;  // const表示该函数不改变类中的成员变量，mutable修饰的成员变量除外 
      ```

- ### 1.4 条款04：对象使用前先初始化

  - 内置变量类型在声明时初始化；
  - 用户创建的类在构造函数用初始化列表初始化（注意：初始化顺序为成员变量的声明顺序有关，与初始化列表如何排列无关）；
  - 不同编译单元的non-local static对象的初始化顺序不确定，尽量使用local static对象代替。

## 2. 构造/析构/赋值

- ### 2.1 条款05：C++会自动编写default构造、拷贝构造、析构、赋值函数

  ```cpp
  //你以为你写了个没有代码的空类
  class Empty{};
  
  // 实际上，C++自动生成了很多函数
  class Empty{
  public:
      Empty() {...}    //默认构造函数
      Empty(const Empty& rhs) {...}    //拷贝构造函数
      ~Empty() {...}     //析构函数

      Empty& operator=(const Empty& rhs) {...}    //赋值函数
  };
  ```

- ### 2.2 条款06：声明为private防止自动生成的函数被使用

  ```cpp
  //把拷贝构造函数和赋值函数声明为private类型，防止被使用
  class Empty {
  public:
      ...
  private:
      ...
      Empty(const Empty&);
      Empty& operator=(const Empty& rhs);
  };
  ```

- ### 2.3 条款07：使用多态特性时，基类的析构函数必须为virtual

  ```cpp
    //一个多态的场景
    class TimeKeeper {   //计时器（基类）
    public:
        TimeKeeper();
        virtual ~TimeKeeper();
        ...
    };
    class AtomicClock: public TimeKeeper {...}    //原子钟
    class WristWatch: public TimeKeeper {...}    //腕表

    //往往这么使用多态特性
    TimeKeeper* ptk = getTimeKeeper();
    ...
    delete ptk;
  ```

  上面是使用多态的一个场景，当`delete ptk`时，因为`ptk`是`TimeKeeper`类型，如果基类析构函数不是`virtual`，那么`ptk`的析构时只调用基类析构函数，析构不正确；使用`virtual ~TimeKeeper();`保证`ptk`析构时调用正确的子类析构函数

  > 如果一个类带有virtual函数，说明这个类有可能用于多态，那么就应该有virtual析构函数。

  > 不要对non-virtual析构函数的类使用多态特性，包括string、vector在内的STL容器。

  > 带有virtual函数的类会生成vtbl (virtual table)用于在运行期间确定具体调用哪个函数，由vptr (virtual table pointer)指向，占用空间。胡乱声明virtual会增加体积。

- ### 2.4 条款08：析构函数不要抛出异常

  - 析构函数抛出异常，程序终止析构，会导致资源没有完全释放。

  ```cpp
      //管理数据库连接的典型场景
      //如果这么写，析构函数有可能抛出异常
      class DBConn {
      public:
          ...
          ~DBConn() {
              db.close();
          }
      private:
          DBConnection db;
      }

      //在析构函数内捕捉异常，不要让析构函数的异常抛出
      class DBConn {
      public:
          ...
          ~DBConn() {
              try { db.close(); }
              catch (...) {
                  ...
              }
          }
      private:
          DBConnection db;
      }
  ```

- ### 2.5 条款09：不在构造和析构函数中使用virtual函数

  - 构造函数在构造时，派生类的成员变量未初始化完毕，virtual函数指向基类；

  - 析构函数在析构时，派生类的成员变量部分析构，virtual函数指向基类。

- ### 2.6 条款10：令 operator= 返回值为 reference to *this

  - 这么做的目的是为了实现连续赋值

  ```cpp
  int x, y, z;
  x = y = z = 1;   //连续赋值

  int& operator=(const int& rhs) {
      ...
      return *this;   //返回左侧对象的引用
  }
  ```

- ### 2.7 条款11：在 operator= 中处理自我赋值

  ```cpp
  //下面代码，如果出现自我赋值，则出错
  Widget& Widget::operator=(const Widget& rhs) {
      delete elem;
      elem = new Bitmap(*rhs.elem);
      return *this;
  }

  //方法1
  Widget& Widget::operator=(const Widget& rhs) {
      if (this == &rhs) { return *this; }

      delete elem;
      elem = new Bitmap(*rhs.elem);
      return *this;
  }

  //方法2
  Widget& Widget::operator=(const Widget& rhs) {
      Bitmap* pOrig = elem;
      elem = new Bitmap(*rhs.elem);
      delete pOrig;
      return *this;
  }
  ```

- ### 2.8 条款12：拷贝构造和赋值函数中不要遗漏成员变量

  - 编译器不检查拷贝构造和赋值函数是否对所有成员变量进行了拷贝；

  - 派生类的拷贝构造和赋值函数记得要先调用基类的拷贝构造和赋值函数。

## 3. 资源管理

- ### 3.1 条款13：使用"资源管理类"管理资源
  
  - 对于 new 出来的对象，某些意外情况下 delete 没有被执行（代码遗漏、鲁棒性低或者出现异常），导致资源泄漏。作者建议使用智能指针管理 new 出来的对象。

  - `auto_ptr`在拷贝或者赋值时，新指针获得资源拥有权，旧指针变为null。

    - ```cpp
      std::auto_ptr<Investment> pInv1(createInvestment());  //pInv1获得资源拥有权
      std::auto_ptr<Investment> pInv2(pInv1);  //pInv2获得资源拥有权，pInv1变null
      pInv1 = pInv2;   //pInv1获得资源拥有权，pInv2变null
       ```
  
  - `shared_ptr`使用计数的方式来确定有哪些指针在使用该资源，计数为0释放资源。它的拷贝和赋值则没有`auto_ptr`的那个问题。

- ### 3.2 条款14：在资源管理类中小心拷贝行为

  - 接3.1，在某些不适用"智能指针"的场景，需要自定义"资源管理类"来管理资源，作者建议需要考虑下这个类的拷贝和赋值。

- ### 3.3 条款15：在"资源管理类"中提供对原始资源的访问

  - 在3.1，我们使用智能指针管理对象，此时指针类型是`std::auto_ptr<Investment>`，但如果有个函数需要类型为`Investment*`的参数怎么办呢？

  - 智能指针有`.get()`函数可以显式转换成原始指针；它们也重载了`operator->`和`operator*`，可以隐式转换。

  - 类似的，自定义的"资源管理类"也需要考虑显示转换和隐式转换，提供对原始资源的访问。

- ### 3.4 条款16：成对使用new和delete时要采取相同形式

  - 简单来说，就是`new`出来的对象要用`delete`释放，`new []`出来的对象要用`delete []`释放。

  - `new`出来的对象如果使用`delete []`释放，将导致未定义的行为；`new []`出来的对象如果用`delete`释放，很可能只释放了数组的第一个元素。

- ### 3.5 条款17：以独立语句将对象置入智能指针

  - ```cpp
    //假设有下面这个函数，它有两个参数，调用该函数时，会做下面三件事：
    //a. 调用new Widget
    //b. 执行shared_ptr的构造
    //c. 执行priority()
    //
    //执行顺序是未定义的，可能为a->b->c，也可能a->c->b等等。注意，当顺序为a->c->b，且priority()抛出异常时，资源泄漏。
    processWidget(std::shared_ptr<Widget>(new Widget), priority());

    //解决方法：别偷懒，写成两句话
    std::shared_ptr<Widget> pW(new Widget);
    processWidget(pW, priority());
    ```
