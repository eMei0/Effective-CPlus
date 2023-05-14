- [6、继承与面向对象设计](#6继承与面向对象设计)
  - [35、考虑 virtual 函数以为的其他选择](#35考虑-virtual-函数以为的其他选择)
    - [藉由 Non-Virtual Interface 手法实现 Template Method](#藉由-non-virtual-interface-手法实现-template-method)
    - [藉由 Function Pointers 实现 Strategy 模式](#藉由-function-pointers-实现-strategy-模式)
    - [藉由 tr1::function 完成 Strategy 模式](#藉由-tr1function-完成-strategy-模式)
  - [36、绝不重新定义继承而来的 non-virtual 函数](#36绝不重新定义继承而来的-non-virtual-函数)
  - [37、绝不重新定义继承而来的缺省参数值](#37绝不重新定义继承而来的缺省参数值)
  - [38、通过复合塑模出 has-a 或”根据某物实现出“](#38通过复合塑模出-has-a-或根据某物实现出)
    - [has-a 关系](#has-a-关系)
    - [根据某物实现出](#根据某物实现出)
  - [39、明智而审慎地使用 private 继承](#39明智而审慎地使用-private-继承)
  - [40、明智而审慎地使用多重继承](#40明智而审慎地使用多重继承)
    - [多继承比单一继承复杂，可能导致歧义](#多继承比单一继承复杂可能导致歧义)
    - [钻石继承 \& virtual inheritance 问题](#钻石继承--virtual-inheritance-问题)
    - [多继承的用武之地](#多继承的用武之地)
- [7、模版与泛型编程](#7模版与泛型编程)
  - [41、了解隐式接口和编译期多态](#41了解隐式接口和编译期多态)
  - [42、了解 typename 的双重意义](#42了解-typename-的双重意义)
  - [43、学习处理模版化基类内的名称](#43学习处理模版化基类内的名称)
  - [44、将与参数无关的代码抽离 templates](#44将与参数无关的代码抽离-templates)
  - [45、运用成员函数模板接受所有兼容类型](#45运用成员函数模板接受所有兼容类型)
  - [46、需要类型转换时请为模板定义非成员函数](#46需要类型转换时请为模板定义非成员函数)

# 6、继承与面向对象设计
 ## 35、考虑 virtual 函数以为的其他选择
### 藉由 Non-Virtual Interface 手法实现 Template Method
```cpp
class GameCharactor {
public:
    int healthValue() const {
        // ...
        int retVal = doHealthValue();
        // ...
        return retVal;
    }

private:
    virtual int doHealthValue() const { /*TODO*/}
};
```
该流派主张 virtual 函数应该总是 private。通过 public non-virtual 函数间接调用 private virtual 函数。它是 ***Template Method*** 设计模式的一种特殊表现形式。
- 优点：提醒你在调用 virtual 函数前，做好事前准备和事后处理。
- 缺点：有时 virtual 不能是 private，无法使用 NVI 手法。
    - derived class 在 virtual 函数中必须调用 base class 的对应函数时，必须为 protected；
    - 具有多态性质的 base classes 的析构函数必须时 public。
### 藉由 Function Pointers 实现 Strategy 模式
```cpp
class GameCharacter;
int defaultHealthCalc(const GameCharactor& gc);
class GameCharactor {
public:
    typedef int (*HealthCalcFunc)(const GameCharactor&);
    explicit GameCharactor(HealthCalcFunc hcf = defaultHealthCalc)
        : healthFunc(hcf)
    {}

private:
    HealthCalcFunc healthFunc;
};

class EvilBadGuy : public GameCharactor {
public:
    explicit EvilBadGuy(HealthCalcFunc hcf = defaultHealthCalc)
        : GameCharactor(hcf)
    {}
};

int loseHealthQuickly(const GameCharactor& gc);
int loseHealthSlowly(const GameCharactor& gc);

EvilBadGuy ebg1(loseHealthQuickly);
EvilBadGuy ebg2(loseHealthSlowly);
```
- 优点：
  - 每个对象都可以拥有自己的计算函数
  - 可在运行时改变。
- 缺点：如果需要 non-public 信息进行精确计算，则需要降低类的封装性（friend 或提供 public 函数）。
### 藉由 tr1::function 完成 Strategy 模式
```cpp
class GameCharacter;
int defaultHealthCalc(const GameCharacter& gc);
class GameCharacter
{
public:
    typedef std::function<int(const GameCharacter&)> HealthCalcFunc;    // 绑定函数签名为int(const GameCharacter&)
    explicit GameCharacter(HealthCalcFunc hcf = defaultHealthCalc)
        : healthFunc(hcf)
    {}
    int healthValue() const
    {
        return healthFunc(*this);
    }
private:
    HealthCalcFunc healthFunc;
};

short calcHealth(const GameCharacter&);

struct HealthCalculator {
    int operator()(const GameCharacter& gc) const
    {}
};

class GameLevel {
public:
    float health(const GameCharacter& gc) const;
};

class EvilBadGuy : public GameCharacter {
public:
    explicit EvilBadGuy(HealthCalcFunc hcf = defaultHealthCalc)
        : GameCharacter(hcf)
    {}
};

class EyeCandyCharacter : public GameCharacter {
public:
    explicit EyeCandyCharacter(HealthCalcFunc hcf = defaultHealthCalc)
        : GameCharacter(hcf)
    {}
};

EvilBadGuy ebg1(calcHealth);    // 使用普通函数
EyeCandyCharacter ecc1(HealthCalculator());   // 使用函数对象
GameLevel currentLevel;
EvilBadGuy ebg2(std::bind(&GameLevel::health, &currentLevel, std::placeholders::_1));   // 使用成员函数
```
std::function 类型可以持有任何与签名式兼容的可调用物，即这个可调用物的参数和返回值可被隐士转换为目标类型。
## 36、绝不重新定义继承而来的 non-virtual 函数
```cpp
class B {
public:
    void mf() {std::cout << "B::mf()" << std::endl;};
};

class D : public B {
public:
    void mf() {std::cout << "D::mf()" << std::endl;};
};

D x;
B* pb = &x;
pb->mf();   // B::mf()
D* pd = &x;
pb->mf();   // D::mf()
```
如果需要重新定义，应定义为 virtual 函数。
## 37、绝不重新定义继承而来的缺省参数值
```cpp
class Shape {
    public:
        enum ShapeColor { Red, Green, Blue };
        virtual void draw(ShapeColor color = Red) = 0;
};

class Rectangle : public Shape {
    public:
        // 指定不同的缺省参数，很糟糕
        // 使用对象或 derived 指针调用时使用的 derived 缺省值
        // 使用 base 指针调用时，使用的时 base 的缺省值，却用的 derived 实现
        void draw(ShapeColor color = Green) {
            std::cout << "Rectangle::draw() color: " << color << std::endl;
        }
};

class Circle : public Shape {
    public:
        // 如果不指定缺省参数，则使用对象调用时，必须指定参数
        // 因为缺省参数是静态绑定，而 virtual 函数是动态绑定
        // 使用指针调用时，可以继承 base 的缺省参数
        void draw(ShapeColor color) {
            std::cout << "Circle::draw() color: " << color << std::endl;
        }
};

    Rectangle rect;
    Rectangle* pr = &rect;
    pr->draw(); // 使用 derived 默认参数
    Shape *ps = &rect;
    ps->draw(); // 使用 base 默认参数以及 derived 实现

    Circle circle;
    circle.draw(Shape::Blue);   // 必须指定参数
    Shape* pc = &circle;
    pc->draw(); // 从基类继承默认参数
```
如果保证 base 和 derived 的缺省值一致，一旦缺省值需要改变，则要同步修改。优化方式是使用 non-virtual interface 方式实现。
## 38、通过复合塑模出 has-a 或”根据某物实现出“
### has-a 关系
```cpp
class Address {};
class PhoneNumber {};
class Person {
public:
private:
    std::string name;
    Address address;
    PhoneNumber phone;
    PhoneNumber faxNumber;
};
```
### 根据某物实现出
```cpp
template <typename T>
class Set {
public:
    bool member(const T item) const;
    void insert(const T item);
    void remove(const T item);
    std::size_t size() const;
private:
    std::list<T> rep;
};
```
## 39、明智而审慎地使用 private 继承
private 继承意味 implemented-in-terms-of 的关系，而不是 is-a 的关系，它与 38 条复合的意义相同。我们尽可能使用复合，必要时才使用 private 继承。
1. 需要重新定义 virtual 函数时；
2. 需要访问 base class 的 protected 成员时；
3. base class 大小为0时，继承可以有 empty base optimization。
```cpp
class Timer {
public:
    explicit Timer(int tickFrequency);
    virtual void onTick() const;
};

class Widget: private Timer {
private:
    virtual void onTick() const override;
};

class Empty {};
class HoldsAnInt: private Empty {
private:
    int x;
};
```

也可以使用复合的方式代替 private 继承，该做法有两个优点：
1. 防止 derived class 重新定义 virtual 函数
2. 将 derived class 编译依存降至最低
```cpp
class Timer {
public:
    explicit Timer(int tickFrequency);
    virtual void onTick() const;
};

class Widget {
private:
    class WidgetTimer : public Timer {
    public:
        virtual void onTick() const override;
    };
    WidgetTimer* pTimer;    // 可以不用包含 WidgetTimer 和 Timer 的头文件
};
```

## 40、明智而审慎地使用多重继承
### 多继承比单一继承复杂，可能导致歧义
```cpp
class BorrowableItem {
public:
    void checkOut();
};

class ElectronicGadget {
private:
    bool checkedOut() const;
};

class MP3Player : public ElectronicGadget, public BorrowableItem {};

MP3Player mp3;
mp3.checkOut(); // error: 'checkOut' is ambiguous
```
### 钻石继承 & virtual inheritance 问题
1. 钻石继承：C++ 缺省做法是让 base class 经由每一条路径复制一次，如果你期望只有一份 base class，那么就要使用 virtual inheritance。
```cpp
class File {);
class InputFile : public File {);   // InputFile is derived from File
class OutputFile : public File {);  // OutputFile is derived from File
class IOFile : public InputFile, public OutputFile {);
```
2. virtual inheritance 的代价
为了避免继承来的成员变量重复，编译器必须提供若干幕后戏法，后果如下：
    - 使用 virtual inheritance 的类的对象的大小会变大；
    - 访问 virtual inheritance 的成员变量的速度会变慢；
    - derived class 初始化时，必须认知 virtual bases；
    - 当新的 derived class 加入继承体系，必须承担其 virtual bases 的初始化责任。
**非必要不使用 virtual bases；如果必须使用 virtual base classes, 尽可能避免在其中放置数据。**
```cpp
class InputFile : virtual public File {);
class OutputFile : virtual public File {);
```
### 多继承的用武之地
```cpp
class IPerson {
public:
    virtual ~IPerson();
    virtual std::string name() const = 0;
    virtual std::string birthDate() const = 0;
};

class DataBaseID {};

class PersonInfo {
public:
    explicit PersonInfo(const DataBaseID& pid);
    virtual ~PersonInfo();
    virtual const char* theName() const;
    virtual const char* theBirthDate() const;
    virtual const char* theDelimOpen() const;
    virtual const char* theDelimClose() const;
};

class CPerson: public IPerson, private PersonInfo {
public:
    explicit CPerson(const DataBaseID& pid): PersonInfo(pid) {};
    virtual std::string name() const { return PersonInfo::theName(); };
    virtual std::string birthDate() const { return PersonInfo::theBirthDate(); };

private:
    const char* theDelimOpen() const { return ""; };
    const char* theDelimClose() const { return ""; };
};
```
多继承的设计方案一定可以某些单继承也可以的方案，但多继承有时的确是最简洁、最易维护、最合理的方案。仔细思考，谨慎使用，别害怕使用。

# 7、模版与泛型编程

## 41、了解隐式接口和编译期多态

- classes 和 template 都支持接口和多态。
- 对 classes 而言接口是显式的，以函数签名为中心。多态则是通过 virtual 函数发生在运行期。
- 对 tamplate 而言，接口是隐式的，奠基于有效表达式。多态则是通过 template 具现化和函数重载解析（function overloading resolution）发生于编译期。

对本节理解不到位，后续补充。

## 42、了解 typename 的双重意义
- 对于嵌套从属名称，C++ 编译器缺省假设不是类型，除非你告诉它是（增加 typename 前缀）。
```cpp
template <typename C>
void print2nd(const C& container)
{
    if (container.size() >= 2)
    {
        typename C::const_iterator iter = container.begin();    // 必须要加 typename
    }
}
```
- typename 不可以出现在 base classes list 及 member initiaization list 内的嵌套从属类型前。
```cpp
template <typename T>
class Derived : public Base<T>::Nested {    // 'typename' cannot appear here
public:
    Derived(int x) : Base<T>::Nested(x) {   // 'typename' cannot appear here
        typename Base<T>::Nested temp;      // OK
    }
};
```
- typename 相关规定在不同的编译器上有不同的实践。这意味着 typename 和“嵌套从属名称”之间的互动，也许会在移植性方面带来麻烦。
## 43、学习处理模版化基类内的名称
- 对于模板化基类，编译器知道其可能被特化，而特化版本可能不提供一般性接口，因此编译器拒绝在模板化基类内查找名称。
```cpp
class CompanyA{
public:
    void sendCleartext(const std::string& msg);
    void sendEncrypted(const std::string& msg);
};

class MsgInfo{};

template<typename Company>
class MsgSender{
public:
    void sendClear(const MsgInfo& info) {
        std::string msg;
        // generate msg from info
        Company c;
        c.sendCleartext(msg);
    }
};

class CompanyZ{
public:
    void sendEncrypted(const std::string& msg);
};

// CompanyZ 的特化版本
template<>
class MsgSender<CompanyZ>{
public:
    void sendSecret(const MsgInfo& info) {}
};

template<typename Company>
class LoggingMsgSender: public MsgSender<Company>{
public:
    void sendClearMsg(const MsgInfo& info) {
        // log start
        sendClear(info);    // compile errors
        // log end
    }
};
```
想让编译器查找模板化基类的名称，可以使用以下三个方法：
1. 使用this->前缀
```cpp
template<typename Company>
class LoggingMsgSender: public MsgSender<Company>{
public:
    void sendClearMsg(const MsgInfo& info) {
        // log start
        this->sendClear(info);    // OK
        // log end
    }
};
```
2. 使用using声明
```cpp
template<typename Company>
class LoggingMsgSender: public MsgSender<Company>{
public:
    using MsgSender<Company>::sendClear;
    void sendClearMsg(const MsgInfo& info) {
        // log start
        sendClear(info);    // OK
        // log end
    }
};
```
3. 明白编译器查找依据，但该方法不推荐，因为它会关闭“virtual 绑定行为”。
```cpp
template<typename Company>
class LoggingMsgSender: public MsgSender<Company>{
public:
    void sendClearMsg(const MsgInfo& info) {
        // log start
        MsgSender<Company>::sendClear(info);    // OK
        // log end
    }
};
```
以上方法都是想编译器承诺“base class template 的任何特化版本都将支持其一般版本所提供的而接口”。如果你的承诺不实现，编译器将会报错，如下：
```cpp
LoggingMsgSender<CompanyZ> zMsgSender;
MsgInfo info;
zMsgSender.sendClearMsg(info);  // compile error
```
## 44、将与参数无关的代码抽离 templates
- Templates 生成多个 classes  和多个函数，所以任何 template 代码都不应该与某个造成代码膨胀的 template 参数产生相依关系。
- 因非类型模板参数而造成的代码膨胀，可以以函数参数或 class 成员变量替换 template 参数来避免。
```cpp
template<typename T, std::size_t n>
class SquareMatrix {
public:
    void invert();
};
squareMatrix<double, 5> sm1;
sm1.invert();
squareMatrix<double, 10> sm2;
sm2.invert();
```
上述代码会具现出两个一样的 invert 函数。我们可以为其建立一个带参数的函数：
```cpp
template<typename T>
class SquareMatrixBase {
protected:
    SquareMatrixBase(std::size_t n, T* pMem): size(n), pData(pMem) {}
    void setDataPtr(T* ptr) { pData = ptr; }
    void invert();
private:
    std::size_t size;
    T* pData;
};

template<typename T, std::size_t n>
class SquareMatrix: private SquareMatrixBase<T> {
public:
    SquareMatrix(): SquareMatrixBase<T>(n, data) {}
private:
    using SquareMatrixBase<T>::invert;
public:
    void invert() { this->invert(); }
private:
    T data[n * n];
};
```
- 因类型模板参数而造成的代码膨胀，可以让带有完全相同二进制表述的具现类型共享实现码来降低。
## 45、运用成员函数模板接受所有兼容类型
```cpp
class Top{};
class Middle: public Top{};
class Bottom: public Middle{};
template<typename T>
class SmartPtr{
public:
    explicit SmartPtr(T* realPtr);
    template<typename U>
    SmartPtr(const SmartPtr<U>& other)
        : ptr(other.get()) {};  // 只有在 U* 能隐式转换为 T* 时，才能通过编译
    T* get() const { return ptr; };
private:
    T* ptr;
};
SmartPtr<Top> pt1 = SmartPtr<Middle>(new Middle);
SmartPtr<Top> pt2 = SmartPtr<Bottom>(new Bottom);
SmartPtr<const Top> cpt1 = pt1;
```
如果你声明 member templates 用于“泛化 copy 构造”或“泛化 assignment 操作”，你还需声明正常的 copy 构造函数和 assignment 操作符。
## 46、需要类型转换时请为模板定义非成员函数
1. 在 template 实参推导过程中从不将隐士类型转换函数纳入考虑。
2. 外部 template 函数只能在 class 内部定义，否则无法连接。
```cpp
template<typename T> class Rational;
template<typename T>
const Rational<T> doMultiply(const Rational<T>& lhs, const Rational<T>& rhs) {};

template<typename T>
class Rational{
public:
    Rational(const T& numerator = 0, const T& denominator = 1);
    const T numerator() const;
    const T denominator() const;
    friend const Rational operator*(const Rational& lhs, const Rational& rhs) { return doMultiply(lhs, rhs); };
};
## 47、请使用 traits classes 表现类型信息
```cpp
struct input_iterator_tag {};
struct output_iterator_tag {};
struct forward_iterator_tag: public input_iterator_tag {};
struct bidirectional_iterator_tag: public forward_iterator_tag {};
struct random_access_iterator_tag: public bidirectional_iterator_tag {};

template<typename T>
class deque {
public:
    class iterator {
    public:
        typedef random_access_iterator_tag iterator_category;
        // ...
    };
    // ...
};

template<typename IterT>
struct iterator_traits {
    typedef typename IterT::iterator_category iterator_category;    // 嵌套从属子类型前要加 typename
    // ...
};

// 以下是 iterator_traits 针对指针迭代器的偏特化版本
template<typename IterT>
struct iterator_traits<IterT*> {
    typedef random_access_iterator_tag iterator_category;
    // ...
};

// 使用重载,在编译期对类型执行测试
template<typename IterT, typename DistT>
void doAdvance(IterT& iter, DistT d, random_access_iterator_tag) {
    iter += d;
}

template<typename IterT, typename DistT>
void doAdvance(IterT& iter, DistT d, bidirectional_iterator_tag) {
    if (d >= 0) {
        while (d--) ++iter;
    } else {
        while (d++) --iter;
    }
}

template<typename IterT, typename DistT>
void doAdvance(IterT& iter, DistT d, input_iterator_tag) {
    if (d < 0) {
        throw std::out_of_range("Negative distance");
    }
    while (d--) ++iter;
}

template<typename IterT, typename DistT>
void Advance(IterT& iter, DistT d) {
    doAdvance(iter, d, iterator_traits<IterT>::iterator_category());
}
```