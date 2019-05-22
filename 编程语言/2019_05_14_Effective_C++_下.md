# Effective C++

作者：LogM

本文原载于 [https://segmentfault.com/u/logm/articles](https://segmentfault.com/u/logm/articles)，不允许转载~

## 7. 模板与泛型

- ### 7.1 条款41：了解隐式接口和编译期多态

  - ```cpp
    //不用模板的写法
    class Widget {
        Widget();
        virtual ~Widget();
        virtual std::size_t size() const;
        virtual void normalize();
        void swap(Widget& other);
        ...
    };

    void doProcess(Widget& w) {
        if (w.size() > 10; && w!= ...) {
            Widget temp(w);
            temp.normalize();
            temp.swap(w);
        }
    }
    //w支持的接口是类型Widget决定的，这称为"显式接口"。
    //Widget类里面的virtual函数是在运行期确定具体调用哪个函数，这称为"运行期多态"。
    ```

  - ```cpp
    //使用模板的写法
    template<typename T>
    void doProcessing(T& w) {
        if (w.size() > 10 && w != ...) {
            T temp(w);
            temp.normalize();
            temp.swap(w);
        }
    }
    //w支持的接口，是由w所参与执行的操作所决定的，比如例子中的w需要支持size()、normalize()、swap()、拷贝构造、不等比较。这称为"隐式接口"。
    //w所参与执行的操作，都有可能导致template的具现化，使函数调用得以成功，具现化发生在编译期。这称为"编译期多态"。
    ```

- ### 7.2 条款42：了解typename的双重意义

  - ```cpp
    //第一重意义
    template<class T> class Widget;
    template<typename T> class Widget;
    //上面两句话效果完全一样
    ```
  
  - ```cpp
    //第二重意义
    //考虑一个例子
    template<typename C>
    void print2nd(const C& container) {
        C::const_iterator* x;   //bad，不加typename被假设为非类型，理由见下面注释
        ...
    }
    //一般，我们认为C::const_iterator指的是某种类型，但是存在一种逗比情况：
    //C是一个类，const_iterator是这个类的int型的成员变量，x是一个int型的变量，那么上面一句话就变成了两个int的相乘。
    //正因为有这种歧义情况的存在，C++假设不加typename的"嵌套从属名称"是非类型。

    //应该这么写
    template<typename C>
    void print2nd(const C& container) {
        typename C::const_iterator* x;   //ok，告诉编译器，C::const_iterator是类型
        ...
    }
    ```

- ### 7.3 条款43：学习处理模板化基类内的名称

  - ```cpp
    //基类
    template<typename T>
    class MsgSender {
    public:
        ...
        void sendClear(const MsgInfo& info);
        ...
    };
    //派生类
    template<typename T>
    class LoggingMsgSender : public MsgSender<T> {
    public:
        void sendClearMsg(const MsgInfo& info) {
            sendClear(info);   //bad，理由见下方注释
        }
    }
    //编译器遇到LoggingMsgSender类时，不知道要继承哪种MsgSender类，所以编译器不知道sendClear这个函数是MsgSender类里继承下来的成员方法，还是类外面的全局的函数。
    //为什么说不同的MsgSender类不一定有sendClear成员方法呢？因为C++允许template的特化，比如我在下面写了一个特化的类，这个特化的类为空类，就没有sendClear成员方法。
    template<>
    class MsgSender<CompanyZ> {  };

    //解决这个问题的方法，本质就是告诉编译器，sendClear函数的来源。具体来说，有三种方法：
    //方法1
    template<typename T>
    class LoggingMsgSender : public MsgSender<T> {
    public:
        void sendClearMsg(const MsgInfo& info) {
            this->sendClear(info);   //ok，告诉编译器，sendClear函数是类内的成员方法
        }
    }
    //方法2
    template<typename T>
    class LoggingMsgSender : public MsgSender<T> {
    using MsgSender<T>::sendClear; //先声明，告诉编译器，如果遇到sendClear函数，则视为类内的成员方法进行编译
    public:
        void sendClearMsg(const MsgInfo& info) {
            sendClear(info);   //ok
        }
    }
    //方法3
    template<typename T>
    class LoggingMsgSender : public MsgSender<T> {
    public:
        void sendClearMsg(const MsgInfo& info) {
            MsgSender<T>::sendClear(info);   //ok，告诉编译器，sendClear函数是类MsgSender<T>内的成员方法
        }
    }
    //方法3不太好的地方是，假如sendClear()是virtual函数，这种写法会把它的多态性破坏；方法1和方法2则不会破坏。
    ```

- ### 7.4 条款44：将与参数无关的代码抽离templates

  - 编译器对template的处理，实际上是对所有可能的template具现出具体代码

  - ```cpp
    //模板类
    template<typename T, std::size_t n>
    class SquareMatrix {
    public:
        ...
        void invert();    //该函数与template无关
    }
    //使用
    SquareMatrix<double, 5> sm1;
    SquareMatrix<double, 10> sm2;
    sm1.invert();
    sm2.invert();
    //这个例子中，invert()函数与template无关，但它被编译器生成了两份，造成重复。
    ```

  - 作者认为将与参数无关的代码抽离templates，可以避免编译器产生这类的重复代码；但我觉得有时候要达到这个目的，会造成代码可读性和编写效率的下降，实际使用时还是要权衡。

- ### 7.5 条款45：运用成员函数模板接受所有兼容类型

  - 假设派生类D继承于基类B，由B具现化的模板类和由D具现化的模板类，并不能相互转换。以代码表述：

  - ```cpp
    class B {...};
    class D : public B {...};

    template<typename T>
    class SmartPtr {
    public:
        SmartPtr(T* realPtr);
        ...
    }

    //使用
    SmartPtr<B> pt1 = SmartPtr<D>(new D);  //bad，SmartPtr<B>与SmartPtr<D>没有继承关系来使得他们相互转换

    //解决方法
    template<typename T>
    class SmartPtr {
    public:
        SmartPtr(T* realPtr);
        template<typename U> SmartPtr(const SmartPtr<U>& other);  //建立一个泛化拷贝构造函数，来解决上面的问题
        ...
    }
    //当然，对于赋值函数也可以这么操作
    ```

- ### 7.6 条款46：需要类型转换时，请为模板定义非成员函数

  - 这条把条款24扩充到模板类上。 

  - ```cpp
    template<typename T>
    class Rational {
    public:
        ...
        Rational(const T& numerator, const T& denominator);
        Rational(sonst T& num);  
        const T numerator() const;
        const T denominator() const;
    }

    const Rational<T> operator* (const Rational<T>& lhs, const Rational<T>& rhs) {
      ...
    }
    //使用
    Rational<int> lhs(1, 9);
    Rational<int> result;
    result = lhs * 2;   //bad，template的推导不考虑隐式类型转换，编译器猜不出T是什么
    result = 2 * lhs;   //bad，template的推导不考虑隐式类型转换，编译器猜不出T是什么

    //解决方法
    template<typename T>
    class Rational {
    public:
        ...
        Rational(const T& numerator, const T& denominator);
        Rational(sonst T& num);  
        const T numerator() const;
        const T denominator() const;

        friend const Rational operator*(const Rational& lhs, const Rational& rhs) {
            //这里要把类外面operator*实现的代码拷贝一份到这里
            ...
        }  
        //在类内声明friend函数，使编译器在类初始化时可以先具现出:
        //"const Rational<int> operator* (const Rational<int>& lhs, const Rational<int>& rhs)"
    };
    const Rational<T> operator* (const Rational<T>& lhs, const Rational<T>& rhs) {
      ...
    }
    //使用
    Rational<int> lhs(1, 9);
    Rational<int> result;
    result = lhs * 2;   //ok，由于friend函数带来的具现化，编译器执行到这里时，具现化好的函数中，已经有满足需要的了，不需要推导T
    result = 2 * lhs;   //ok，由于friend函数带来的具现化，编译器执行到这里时，具现化好的函数中，已经有满足需要的了，不需要推导T
    ```

- ### 7.7 条款47：使用traits classes表现类型信息

  - STL中广泛使用traits classes来标记容器属于哪一类容器（比如"可随机访问容器"：vector、deque等）

- ###  7.8 条款48：认识template元编程（TMP）

  - 所谓元编程，是执行于编译器内的程序，C++以template实现元编程。

  - 优点：a. 完成一些以前不可能完成的任务；b. 将工作从运行期转移到编译期（比如之前在运行期才找到的错误可以在编译期找到）。

  - 缺点：编译时间变长。

  - TMP不同于"正常化的"C++，还没有完全被C++标准支持，普通用户可以暂时不用了解。

## 8. 定制new和delete

- 8.1 条款49：了解new-handler的行为

  - new-handler相当于new的异常处理函数。new申请内存发生异常，调用new-handler处理，处理完了之后继续尝试new，如果还是出错，继续调用new-handler，以此反复。

  - ```cpp
    //标准库对于new-handler是这么写的：
    //new_handler被定义成一个函数指针
    //set_new_handler函数的参数是个指针，指向new无法分配足够内存时应该调用的函数；
    //set_new_handler函数的返回值是个指针，指向set_new_handler之前设置的new_handler函数
    namespace std {
        typedef void (*new_handler)();
        new_handler set_new_handler(new_handler p) throw();
    }

    //可以这样使用set_new_handler
    void oufOfMem()
    {
        std::cerr << "Unalbe to new\n";
        std::abort();
    }
    int main()
    {
        std::set_new_handler(outOfMem);
        int * pBigArray = new int[10000000000L];
    }
    ```

  - 一般来说，new-handler函数会做以下事情的某几件：
    - a. 让更多内存可被使用。使得再次尝试new能成功。
    - b. 安装另一个new-handler。使用更强力的new-handler处理。
    - c. 卸载new-handler。实在不行，卸载new-handler使new抛出异常。
    - d. 抛出bad_alloc的异常。
    - e. 退出程序。abort()或exit()。

- ### 8.2 条款50：了解new和delete的合理替换时机

  - 这部分讲的是：如果你觉得编译器自带的new和delete不好用，应该怎么写一个自定义的函数替换。

  - 作者自己也提到了，要自定义new和delete不是简单写个函数就好了，内存的申请和释放会涉及到计算机体系架构中比较底层的东西，比如"内存对齐"。所以重头开始写new和delete是比较复杂的，可以买现成的商业产品，或者在开源代码上修改。

- ### 8.3 条款51：编写new和delete时需要固守常规

  - 条款50已经说明了，自定义new和delete比较复杂，非必要不建议重写。

  - 这部分讲了自定义new和delete时需要注意的事项：
    - operator new应该内含一个无穷循环，并在其中尝试分配内存，分配不了，调用new-handler。
    - operator new对0 bytes的内存申请，也需要正常返回指针。
    - operator delete应该在收到null指针时不做任何事。

- ### 8.4 条款52：写了placement new也要写placement delete

  - ```cpp
    Widget* pw = new Widget
    //这句话中总共有2个函数被调用：1.operator new分配内存；2.Widget的构造函数
    //当调用Widget构造函数发生异常时，需要delete掉第一步new出来的内存
    //如果使用C++自带的new和delete，这个情况不需要用户考虑；如果是自定义new和delete，需要用户考虑
    //因为普通用户很少自定义new和delete，这里不再深入。
    ```

## 9. 杂项

- ### 9.1 条款53：不要轻易忽视编译器的警告

  - 也不要依赖编译器给你指出错误，因为不同的编译器对错误的敏感度是不同的。

- ### 9.2 条款54：让自己首席包括TR1在内的标准程序库

  - C++的一些扩展特性会在TR1，虽然这些特性随着C++标准版本的更新逐渐合并到标准中。

- ### 9.3 条款55：让自己熟悉Boost

  - Boost是一个C++开发者社群，由C++委员会创建，它是C++新特性的试验场，TR1的许多扩展特性是从Boost提交的。