- [6、继承与面向对象设计](#6继承与面向对象设计)
  - [35、考虑 virtual 函数以为的其他选择](#35考虑-virtual-函数以为的其他选择)
    - [藉由 Non-Virtual Interface 手法实现 Template Method](#藉由-non-virtual-interface-手法实现-template-method)
    - [藉由 Function Pointers 实现 Strategy 模式](#藉由-function-pointers-实现-strategy-模式)
    - [藉由 tr1::function 完成 Strategy 模式](#藉由-tr1function-完成-strategy-模式)
  - [36、绝不重新定义继承而来的 non-virtual 函数](#36绝不重新定义继承而来的-non-virtual-函数)
  - [37、绝不重新定义继承而来的缺省参数值](#37绝不重新定义继承而来的缺省参数值)

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
