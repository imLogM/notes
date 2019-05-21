# Effective C++

作者：LogM

本文原载于 [https://segmentfault.com/u/logm/articles](https://segmentfault.com/u/logm/articles)，不允许转载~

## 4. 设计与声明

- ### 4.1 条款18：让接口不易被误用

  - 简单来说，就是考虑用户可能的误用行为，在代码中规避。比如工厂函数返回值为"智能指针"类型，避免用户使用一般指针管理资源带来的资源泄漏风险；比如将 operator* 的返回类型设为const，避免用户写出"a\*b=c"这样的代码。

- ### 4.2 条款19：像设计type一样设计class

  - 作者的意思是，设计class要考虑很多细节，尽量使设计出来的class像C++内置类型一样有极大的可用性和鲁棒性。

- ### 4.3 条款20：宁以 pass-by-reference-to-const 替换 pass-by-value
  
  - 原因：a. 参数使用引用传递，不需要构造新对象，比较快；b. 避免对象切割问题。

  - > 对象切割（slicing）：当派生类对象以 by-value 方式传递参数并被视为基类对象，基类的拷贝构造函数在构造时会把派生类特有的性质抹除。

  - > "引用"在编译器底层的实现就是指针（指针的本质是int类型的变量），所以在以下场景，"引用"并不一定比pass-by-value快：a. C++内置类型；b. STL的迭代器（底层实现是指针）；c. 函数对象（底层实现是指针）。

- ### 4.4 条款21：不要让函数返回值是指向局部变量的引用或指针

  - 原因应该很容易理解：局部变量在函数调用结束后就销毁了，那么这个函数返回的引用和指针指向的内存已经无效了。

- ### 4.5 条款22：将成员变量声明为 private

  - 一致性：成员变量为 private，则用户想访问成员变量必须通过成员函数，所以用户就不用试着记住是不是要加括号，因为成员函数都要加括号。

  - 安全性：通过成员函数控制用户对成员变量的读写权限。

  - 封装性：class的版本发生变化，但提供给用户的API还是可以保持不变。

- ### 4.6 条款23：宁以 non-menber、non-friend 替换 member 函数

  - 使用场景如下代码所示。作者倾向于non-member、non-friend的理由是：它们无法访问private的成员变量，在封装性上更好，编译时候的依赖程度也低。

  - 作者的说法有一定道理，但我不完全同意作者的观点：
    - a. member函数可以访问private的成员变量，并不意味着用户就可以接触到private的成员变量，你在写代码的时候不让这个member函数访问private的成员变量不就可以了？（此时问题变成了：如何确保写代码的人不在这个函数中滥用private的成员变量）

    - b. 有些情况，使用non-member、non-friend函数会降低代码接口的一致性。作者的解决思路是把non-member、non-friend函数放在和类同一个namespace下，我想了想，这么做一致性还是不如直接写member函数。

  - ```cpp
    class WebBrowser {
    public:
        ...
        void clearCache();
        void clearHistory();
        void clearCookies();
        ...
    }

    //假如现在要写一个函数clearEverything()，作用是同时清理cache、history、cookies。

    //使用member函数的情况
    class WebBrowser {
    public:
        ...
        void clearEverything();
        ...
    }
    void WebBrowser::clearEverything() {
        clearCache();
        clearHistory();
        clearCookies();
    }

    //使用non-member函数的情况
    void clearBrower(WebBrowser& wb) {
        wb.clearCache();
        wb.clearHistory();
        wb.clearCookies();
    }
    ```

- ### 4.7 条款24：若所有参数都需要类型转换，请把这个函数写成 non-member 函数

  - ```cpp
    //第一种情况：乘法函数为member函数
    class Rational {
    public:
        ...
        Rational(int numerator, int denominator);
        Rational(int num);  //这个构造函数使得该类支持从int到Rational的类型转换。如果前面加explict则说明不支持隐式类型转换仅支持显式转换，现在没加，支持隐式转换
        const Rational operator* (const Rational& rhs) const;
    }

    //使用
    Rational lhs(1, 9);
    Rational result;
    result = lhs * 2;   //ok，2不是Rational类型，但可以发生隐式类型转换
    result = 2 * lhs;   //bad，2不是Rational类型
    ```
  
  - ```cpp
    //第二种情况：乘法函数为non-member函数
    class Rational {
    public:
        ...
        Rational(int numerator, int denominator);
        Rational(int num);  //这个构造函数使得该类支持从int到Rational的类型转换
    }

    const Rational operator* (const Rational& lhs, const Rational& rhs) {
      ...
    }

    //使用
    Rational lhs(1, 9);
    Rational result;
    result = lhs * 2;   //ok，2不是Rational类型，但可以发生隐式类型转换
    result = 2 * lhs;   //ok，2不是Rational类型，但可以发生隐式类型转换
    ```

- ### 4.8 条款25：考虑写出一个不抛异常的swap函数

  - STL库中`std::swap`的典型实现如下代码，这个实现比较平淡。对于某些类，写一个模板特化的swap执行效率会更高。作者介绍了怎么自己写`std::swap`的特化版本，这边就不展开了。

  - ```cpp
    namespace std {   //平淡的std::swap实现
        template<typename T>
        void swap(T& a, T& b) {
            T temp(a);
            a = b;
            b = temp;
        }
    }
    ```

## 5. 实现

- ### 5.1 条款26：尽可能延后变量定义式出现的时间

  - ```cpp
    std::string encryptPassword(const std::string& password) {
        using namespace std;
        string encrypted;   //bad，过早定义，如果下一句出异常，这个变量其实没有用到，导致无谓的构造和析构
        if (password.length() < MinimumPasswordLength) {
            throw logic_error("Password is too short");
        }
        ...   //加密过程
        return encrypted;
    }

    std::string encryptPassword(const std::string& password) {
        using namespace std;
        if (password.length() < MinimumPasswordLength) {
            throw logic_error("Password is too short");
        }

        string encrypted;   //ok，要用到时再定义
        ...  //加密过程
        return encrypted;
    }
    ```

  - ```cpp
    //循环内的变量怎么定义？

    //方法A：定义于循环外
    Widget w;
    for (int i=0; i<n; ++i) {
        w = ...;
        ...
    }

    //方法B：定义于循环内
    for (int i=0; i<n; ++i) {
        Widget w(...);
        ...
    }

    //方法A：1个构造函数、1个析构函数、n个赋值操作，变量w作用域覆盖到循环以外
    //方法B：n个构造函数、n个析构函数，变量w作用域仅在循环内
    //具体使用哪个视情况而定。

    //我自己想了一个非常小众的写法，虽然可以兼得方法A和方法B的长处，但是代码可读性降低
    for (int i=0, Widget w; i<n; ++i) {
        w = ...;
        ...
    }
    ```

- ### 5.2 条款27：尽量少做转型动作

  - C++的新式转型：
    - a. `const_cast<...>(...)`。移除对象的常量性。
    - b. `dynamic_cast<...>(...)`。在继承体系中进行安全的向下转型，效率低。
    - c. `reinterpret_cast<...>(...)`。低级转型，结果取决于编译器，不可移植。
    - d. `static_cast<...>(...)`。强迫执行隐式转换，类似于C中的旧式转换。

  - 作者提醒，只有明确知道转型后的结果是什么时才使用转型，自以为是地猜测转型的结果往往导致错误。

- ### 5.3 条款28：避免返回值是指向 private 成员变量的引用或者指针
  
  - 很容易理解，如果成员函数返回值是指向 private 成员变量的引用或者指针，那么成员变量的 private 不再具有意义，它实际能被修改。

  - 解决方法是将返回的引用或指针设为const类型，禁止修改。

  - 有些特殊情况下，返回的引用或指针指向的成员变量被销毁，导致引用和指针无效（dangling handles），这种情况要在代码中避免。

- ### 5.4 条款29：保证"异常安全"

  - 异常安全的函数，即使在发生异常的时候也能保证不泄漏资源不破坏数据结构，可以分为三种：
    - a. 基本型：保证资源不泄露，保证数据结构不破坏，但异常发生后程序中的对象的状态是未知的；
    - b. 强烈型：在基本型的基础上，保证如果出现异常，程序中的对象的状态会回复到"调用该函数前"的状态；
    - c. 不抛异常型：承诺所有异常都能被处理，不抛出异常。

  - 实际使用时，要综合考虑效率和安全性来确定使用哪一种类型的异常安全。

- ### 5.4 条款30：透彻理解inline函数

  - inline函数：编译器用inline函数的本体代码替换程序中的每一次调用。因此，滥用inline函数将导致程序体积过大，进而降低高速缓存命中率等导致效率降低。

  - inline函数的实现应该写在`.h`头文件中，因为大多数编译器是在编译期处理inline函数的。（与template类似，template的声明和实现也应该放在头文件中，但这不意味着template就一定是inline）

  - virtual函数不能使用inline，因为virtual是运行期才能确定的。

  - inline的缺点：a. inline函数无法随着程序库的升级而升级，必须重新编译；b. inline函数的调试很麻烦。

  - 什么时候用inline：当整个程序的大量运行时间耗费在某段代码的调用过程上时。

  - 即使你写成inline函数，编译器也有权力不编译为inline形式。

- ### 5.5 条款31：将文件间的编译依存关系降到最低

  - 原因：当改动一个文件后，不需要整个工程都重新编译。

  - 常见的"把类的定义和声明放在两个文件中"，就是减少编译依存关系的一个例子。

## 6. 继承与面向对象

- ### 6.1 条款32：确定你的public继承是is-a关系

  - public继承的子类对象需要保证可以被视作父类对象来调用函数。

  - ```cpp
    class Person {...};
    class Student : public Person {...};

    void eat(const Person& p);
    void study(const Student& s);
    eat(p);   // ok
    eat(s);   // ok，Student可以视作Person调用函数
    study(s);   //ok
    study(p);   //error，Person不能视作Student
    ```

- ### 6.2 条款33：避免遮掩继承而来的名称

  - ```cpp
    int x;  //global变量
    void someFunc() {
        double x;  //local变量
        std::cin >> x;  //local变量的x遮掩了global变量的x，实际起作用的是local变量的x
    }
    ```
  
  - ```cpp
    //定义基类
    class Base {
    private:
        int x;
    public:
        virtual void mf1() = 0;
        virtual void mf1(int);
        virtual void mf2();
        void mf3();
        void mf3(double);
        ...
    };
    //定义派生类
    class Derived : public Base {
    public:
        virtual void mf1();
        void mf3();
        void mf4();
        ...
    };

    //使用
    Derived d;
    int x;
    ...
    d.mf1();    //ok，调用Derived::mf1
    d.mf1(x);   //bad，因为Derived::mf1遮掩了Base::mf1
    d.mf2();    //ok，调用Base::mf2
    d.mf3();    //ok，调用Derived::mf3
    d.mf3(x);   //bad，因为Derived::mf3遮掩了Base::mf3
    ```

  - ```cpp
    //解决方法1
    //定义派生类
    class Derived : public Base {
    public:
        using Base::mf1;    //让基类中名为mf1和mf3的所有东西在此作用域内可见
        using Base::mf3;
        virtual void mf1();
        void mf3();
        void mf4();
        ...
    };

    //使用
    Derived d;
    int x;
    ...
    d.mf1();    //ok，调用Derived::mf1
    d.mf1(x);   //ok，调用Base::mf1
    d.mf2();    //ok，调用Base::mf2
    d.mf3();    //ok，调用Derived::mf3
    d.mf3(x);   //ok，调用Base::mf3
    ```

  - ```cpp
    //解决方法2
    //定义基类
    class Base {
    public:
        virtual void mf1() = 0;
        virtual void mf1(int);
        ...
    };
    //定义派生类
    class Derived : public Base {
    public:
        virtual void mf1() { Base::mf1(); }  // 转交函数
        ...
    };

    //使用
    Derived d;
    int x;
    ...
    d.mf1();    //ok，调用Derived::mf1
    d.mf1(x);   //bad，因为Base::mf1被遮掩，且Base::mf1(int)没有被转交
    ```

- ### 6.3 条款34：区分接口继承和实现继承

  - 如下代码所示，有3类继承关系：
    - 纯虚函数的继承，目的是让派生类只继承函数接口。派生类必须实现该接口。
    - 非纯虚函数的继承，目的是让派生类继承函数接口和缺省实现。派生类可以实现该接口，也可以选择使用基类的缺省实现。
    - 非虚函数的继承，目的是让派生类继承函数接口和强制实现。派生类不应该另外实现该接口，否则发生上一个条款所述的遮掩行为。

  - ```cpp
    class Shape {
    public:
        virtual void draw() const = 0;
        virtual void error(const std::string& msg);
        int objectID() const;
        ...
    };

    class Rectangle : public Shape {...};
    class Ellipse : public Shape {...};
    ```

- ### 6.4 条款35：考虑virtual函数之外的其他选择

  - ```cpp
    //假设你正在写一个游戏软件，每个游戏人物都应该有"健康"这个属性
    class GameCharacter {
    public:
        virtual int healthValue() const;  //返回游戏人物的健康状态
        ...
    };
    ```
  
  - 上面的写法没有问题，基类提供了接口和一个缺省的实现，派生类可以选择是否重写这个函数。但是，作者也提醒，还有一些其它的写法：

    - ```cpp
      //方法1：借由Non-Virtual Interface（NVI）手法实现Template Method模式
      class GameCharacter {
      public:
          int healthValue() const {
              ... //在开始主函数前，可以做一些额外的工作
              int retVal = doHealthValue();   //这个函数可以被子函数重写
              ... //在结束主函数后，可以做一些额外的工作
              return retVal;
          }
          ...
      private:
          virtual int doHealthValue() const {
              ...
          }
      };
      ```

    - ```cpp
      //方法2：借由函数指针实现Strategy模式
      //这种方法允许同一个派生类的不同对象使用不同的函数计算健康状态
      class GameCharacter {
      public:
          typedef int (*HealthCalcFunc)(const GameCharacter&);
          explicit GameCharacter(HealthCalcFunc hcf = defaultHealthCalc)
            : healthFunc(hcf) {

          }
          int healthValue const {
              return healthFunc(*this);
          }
          ...
      private:
          HealthCalcFunc healthFunc;
      };
      ```

    - ```cpp
      //方法3：借由tr1::function完成Strategy模式
      //这种方法比函数指针更自由，它可以是函数指针，也可以是任何可以被调用的东西
      class GameCharacter {
      public:
          typedef std::tr1::function<int (const GameCharacter&)> HealthCalcFunc;
          explicit GameCharacter(HealthCalcFunc hcf = defaultHealthCalc)
            : healthFunc(hcf) {

          }
          int healthValue const {
              return healthFunc(*this);
          }
          ...
      private:
          HealthCalcFunc healthFunc;
      }
      ```

    - ```cpp
      //方法4：古典的Strategy模式
      //HealthCalcFunc来自另一个继承体系
      class GameCharacter;
      class HealthCalcFunc {
      public:
          ...
          virtual int calc(const GameCharacter& gc) const {...}
          ...
      };
      HealthCalcFunc defaultHealthCalc;

      class GameCharacter {
      public:
          explicit GameCharacter(HealthCalcFun* phcf = &defaultHealthCalc)
              : pHealthCalc(phcf) {

          }
          int healthValue() const {
              return pHealthCalc->calc(*this);
          }
          ...
      private:
          HealthCalcFunc* pHealthCalc;
      };
      ```

- ### 6.5 条款36：绝不重新定义继承而来的non-virtual函数

  - 我怀疑作者这条讲重复了，条款33讲"遮掩"的时候就讲到了这个内容

  - ```cpp
    //基类
    class B {
    public:
        void mf();
        ...
    };
    //派生类
    class D {
    public:
        void mf();   //遮掩B::mf
        ...
    };
    //使用
    D x;
    B* pB = &d;   //调用B::mf()
    pB->mf();
    D* pD = &d;   //调用D::mf()
    pD->mf();
    //如果B::mf()是virtual函数，则上面两处都是调用D::mf()
    //所以也就可以解释，条款7所述的"使用多态时，基类的析构必须为virtual"
    ```

- ### 6.6 条款37：绝不重新定义继承而来的缺省参数值

  - 条款36说了non-virtual不要重新定义，所以在派生类中能重新定义的是virtual函数。
  - virtual函数是动态绑定，而缺省参数值是静态绑定，这就会带来问题。

  - ```cpp
    //基类
    class Shape {
    public:
         enum ShapeColor {Red, Green, Blue};
         virtual void draw(ShapeColor color=Red) const = 0;
    };
    //派生类
    class Rectangle : public Shape {
    public:
        virtual void draw(ShapeColor color=Green) const;
        ...
    };
    //使用
    Shape* pR = new Rectangle;  //请注意类型是Shape*
    pR->draw();   //调用Rectangle::draw(Shape::Red)，因为缺省值是在编译期间静态绑定的，而pR的静态类型为Shape*，是基类
    ```

  - 解决方法是在条款35中找一种替代设计，比如NVI。

  - ```cpp
    //基类
    class Shape {
    public:
        enum ShapeColor {Red, Green, Blue};
        void draw(ShapeColor color=Red) const {
            doDraw(color);
        }
        ...
    private:
        virtual void doDraw(ShapeColor color) const = 0;
    };
    //派生类
    class Rectangle : public Shape {
    public:
        ...
    private:
        virtual void doDraw(ShapeColor color) const {
            ...
        }
        ...
    };
    ```

- ### 6.7 条款38：确保"复合"是has-a或is-implemented-in-terms-of关系
  
  - 复合（composition）是类之间的一种关系，不同于public继承，它代表has-a或"根据某物实现出"这样一种关系。

  - ```cpp
    class Address {...};
    class PhoneNumber {...};
    class Person {
    public:
        ...
    private:
        std::string name;
        Address address;
        PhoneNumber voiceNumber;
        PhoneNumber faxNumber;
    };
    ```

- ### 6.8 条款39：明智而审慎地使用private继承

  - private继承会将基类中继承而来的所有成员变为private成员。

  - 如下代码所示，private继承的关系更像是"复合"（Student类中带有Person类的一些功能），只有软件实现层面的意义，不具有设计层面上的意义。尽可能使用"复合"替代private继承。

  - ```cpp
    class Person {...};
    class Student : private Person {...};
    void eat(const Person& p);

    Person p;
    Student s;
    eat(p);   //ok
    eat(s);   //bad，Student不能被视为Person
    ```

- ### 6.9 条款40：明智而审慎地使用多重继承

  - 多重继承会发生"成员函数或成员变量重名"、"钻石型继承"等问题，对于这种情况建议使用virtual public继承，但这会带来空间和效率的损失。

  - 多重继承的复杂性，导致一般不会使用。只有virtual base classes（也就是继承多个接口）才最有实用价值，它不一定需要virtual public继承。
