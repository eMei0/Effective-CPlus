[toc]
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
