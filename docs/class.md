# 如何写一个类

## cheat sheet

```cpp
// ## Classes

class T {                   // A new type
private:                    // Section accessible only to T's member functions
protected:                  // Also accessible to classes derived from T
public:                     // Accessible to all
    int x;                  // Member data
    void f();               // Member function
    void g() {return;}      // Inline member function
    void h() const;         // Does not modify any data members
    int operator+(int y);   // t+y means t.operator+(y)
    int operator-();        // -t means t.operator-()
    T(): x(1) {}            // Constructor with initialization list
    T(const T& t): x(t.x) {}// Copy constructor
    T& operator=(const T& t)
    {x=t.x; return *this; } // Assignment operator
    ~T();                   // Destructor (automatic cleanup routine)
    explicit T(int a);      // Allow t=T(3) but not t=3
    T(float x): T((int)x) {}// Delegate constructor to T(int)
    operator int() const
    {return x;}             // Allows int(t)
    friend void i();        // Global function i() has private access
    friend class U;         // Members of class U have private access
    static int y;           // Data shared by all T objects
    static void l();        // Shared code.  May access y but not x
    class Z {};             // Nested class T::Z
    typedef int V;          // T::V means int
};
void T::f() {               // Code for member function f of class T
    this->x = x;}           // this is address of self (means x=x;)
int T::y = 2;               // Initialization of static member (required)
T::l();                     // Call to static member
T t;                        // Create object t implicit call constructor
t.f();                      // Call method f on object t

struct T {                  // Equivalent to: class T { public:
    virtual void i();         // May be overridden at run time by derived class
    virtual void g()=0; };    // Must be overridden (pure virtual)
class U: public T {         // Derived class U inherits all members of base T
public:
    void g(int) override; };  // Override method g
class V: private T {};      // Inherited members of T become private
class W: public T, public U {};
// Multiple inheritance
class X: public virtual T {};
// Classes derived from X have base T directly

// All classes have a default copy constructor, assignment operator, and destructor, which perform the
// corresponding operations on each data member and each base class as shown above. There is also a default no-argument
// constructor (required to create arrays) if the class has no constructors. Constructors, assignment, and
// destructors do not inherit.
```

## Sample

```cpp
class Sample {
    Sample() {};
    virtual ~Sample() {};
}
```



## 构造函数与析构函数

构造函数只进行没有实际意义（trivial）的初始化工作，例如将指针初始化为 `NULL`，将变量初始化为 `0` 或者 `-1`。不允许在构造函数进行有实际意义（non-trivial）的初始化操作，如果需要，定义单独的 `int init()` 方法并增加一个 `is_inited_`变量标识对象是否已经初始化成功。这是因为，如果对象构造失败，可能会出现不确定的状态。

每个类（包括接口类）都要求定义构造函数，即使该类没有任何成员变量，也需要定义一个空的默认构造函数。这是因为，如果没有定义任何构造函数，编译器会自动生成默认构造函数，这样往往会产生一些副作用，例如将某些成员变量初始化成随机值。

每个类（包括接口类）都要求定义析构函数，即使该类没有任何成员变量，也需要定义一个空的析构函数。另外，如果没有特殊原因（性能特别关键，不会被继承且不包含虚函数），都应该把类的析构函数声明为 `virtual`。

## explicit 关键字

对单参数构造函数使用C++关键字 `explicit`。

由于 C++ 的隐式转换机制，只有一个参数的构造函数容易被错误的调用。例如，假设程序定义了构造函数 `ObFoo::ObFoo(ObString name)`和普通函数`void f(ObFoo foo)`，当调用函数 `void f(ObFoo foo)` 并且误传入了 `ObString` 对象作为参数时，`ObString` 对象会因为隐式转换机制，被构造函数 `ObFoo::ObFoo(ObString name)` 自动转换为一个 `ObFoo` 临时对象传给调用函数，导致程序被错误的执行。因此，允许构造函数使用隐式转换机制，可能会带来潜在的 bug。

## *拷贝构造函数

**原则上不得使用拷贝构造函数（已经定义使用的基础类除外）。**如果必须违反，请事先征得小组负责人的同意，并详细注释原因。另外，不需要拷贝构造函数的类应使用 `DISALLOW_COPY_AND_ASSIGN` 宏（`ob_define.h`），接口类可以例外。

```cpp
#define DISALLOW_COPY_AND_ASSIGN(type_name) \
 type_name(const type_name&);             \
 void operator=(const type_name&)
 
class ObFoo
{
public:
 ObFoo();
 ~ObFoo();
 
private:
 DISALLOW_COPY_AND_ASSIGN(ObFoo);
};
```

## reuse & reset & clear

**`reset`** **方法****用于重置对象，****`reuse`** **方法****用于重用对象，禁止使用** **`clear`** **方法**。说明如下：

1. `reset` 和 `reuse` 都用于对象的重用。这种重用往往是因为某些关键类的对象分配内存和构造太耗时，为了优化性能而引入的。
2. `reset` 的含义是把对象的状态恢复成构造函数或者 `init` 方法执行以后的初始状态。参考 `ObRow::reset`；
3. 除了 `reset` 以外的其它情况采用 `reuse`。与 `reset` 不同的是，`reuse` 往往不释放某些重新申请很耗时的资源，如内存等。参考 `PageArena::reuse`。
4. `clear` 广泛用于 C++ STL 的容器类中，往往表示把容器类的 `size` 清零，但是并不清空内部的对象或者释放内存。`clear` 和 `reset`/`reuse` 的区别很微妙，为了简化理解，禁止使用 `clear`，已使用的逐步去除。

## 成员初始化

**必须初始化所有成员，且成员变量初始化顺序和定义顺序保持一致**。

每个类的构造函数、`init` 方法、`reset`/`reuse` 方法都可能对类执行一些初始化操作，需要确保所有的成员都已经初始化。如果构造函数只初始化部分成员，那么，一定要将 `is_inited_` 变量设置为 `false`，由 `init` 方法继续完成其它成员的初始化工作。`struct` 类型的成员可以通过 `reset` 方法初始化。但如果初始化仅仅是将 `struct` 成员清零，也可以使用 `memset` 初始化；class 类型的成员可以通过 `init`/`reset`/`reuse` 等方法初始化。

成员变量的初始化顺序需要和定义顺序保持一致，这样的好处是很容易发现是否忘记初始化了哪些成员。

## 结构体和类

仅当只有数据时使用 `struct`，其他一概使用 `class`。

`struct` 被用在仅包含数据的消极对象上，可能包括有关联的常量，以及 `reset`/`is_valid`，序列化/反序列化这几个通用函数。如果需要更多的函数功能，class 更加合适。如果不确定的话，直接使用`class`。

如果与 STL 结合，对于仿函数（functor）和萃取（traits）可以不用 `class` 而是使用 `struct`。

需要注意的是，`class` 内部的数据成员只能定义为私有的（静态成员可以例外），并通过存取函数 `get_xxx`和 `set_xxx` 进行访问。

## 通用函数

每个类包含的通用函数都必须采用标准原型，序列化/反序列化函数必须使用宏实现。

每个类包含的通用函数包括：`init`、`destroy`、`reuse`、`reset`、`deep_copy`、`shallow_copy`、`to_string`、`is_valid`。这些函数的原型如下：

```cpp
class ObFoo
{
public:
int init(init_param_list);
bool is_inited();
void destroy();
 
void reset();
void reuse();
 
int deep_copy(const ObFoo &src);
int shallow_copy(const ObFoo &src);
 
bool is_valid();
 
int64_t to_string(char *buf, const int64_t buf_len) const;
NEED_SERIALIZE_AND_DESERIALIZE;
};
```

注意，`to_string` 总是会在末尾补 `’\0’`，且函数返回实际打印的字节长度（不包括 `’\0’`）。内部通过调用 `databuff_printf` 相关函数实现，具体请参考 `common/ob_print_utils.h`。

序列化和反序列化函数需要通过宏来实现，举个例子：

```cpp
class ObSort
{
public:
NEED_SERIALIZE_AND_DESERIALIZE;
private:
common::ObSArray<ObSortColumn> sort_columns_;
int64_t mem_size_limit_;
int64_t sort_row_count_;
};
```

对于类 `ObSort`，需要序列化的三个域是 `sort_columns_`、`mem_size_limit_`、`sort_row_count_`。在 `ob_sort.cpp` 里面只需要写上：

```cpp
DEFINE_SERIALIZE_AND_DESERIALIZE(ObSort, sort_columns_, mem_size_limit_, sort_row_count_);
```

就能完成序列化和反序列化已经计算序列化以后得长度的三个函数的实现。

结构体的通用函数与类的通用函数相同。

## 常用宏

为了方便编码，在 OceanBaes 中可以使用一些已经定义过的宏，但并不建议新增宏，如果确实有新增宏的必要，请和小组负责人确认后再行添加。

以下是一些常用的宏：

1. `OB_SUCC`

通常用于判断返回值是否为 `OB_SUCCESS`，等价于 `OB_SUCCESS == (ret = func())`，注意使用 `OB_SUCC` 时需要在函数内前置定义 `ret`，例如下面的写法，

```cpp
ret = OB_SUCCESS;
if (OB_SUCC(func())) {
  // do something
}
```

1. `OB_FAIL`

通常用于判断返回值是否不为 `OB_SUCCESS`，等价于 `OB_SUCCESS != (ret = func())`，注意使用 `OB_FAIL` 时需要在函数内前置定义 `ret`，例如下面的写法，

```cpp
ret = OB_SUCCESS;
if (OB_FAIL(func())) {
  // do something
}
```

1. `OB_ISNULL`

通常用于判断指针是否为空，等价于 `nullptr ==`，例如下面的写法，

```
if (OB_ISNULL(ptr)) {
  // do something
}
```

1. `OB_NOT_NULL`

通常用于判断指针是否不为空，等价于 `nullptr !=` ，例如下面的写法，

```
if (OB_NOT_NULL(ptr)) {
  // do something
}
```

1. `IS_INIT`

通常用于判断类是否完成了初始化，等价于 `is_inited_`，注意在类中需要存在成员 `is_inited_`，例如下面的写法，

```
if (IS_INIT) {
  // do something
}
```

1. `IS_NOT_INIT`

通常用于判断类是否完成了初始化，等价于 `!is_inited_`，注意在类中需要存在成员 `is_inited_`，例如下面的写法，

```
if (IS_NOT_INIT) {
  // do something
}
```

1. `REACH_TIME_INTERVAL`

用于判断是否超过了某个时间间隔（单位为毫秒），注意在宏内部会有一个静态变量记录时间，所以对时间的判断是全局的，通常用于控制日志输出频率，例如下面的写法，会让系统在超过 1s 间隔后，做一些动作。

```
if (REACH_TIME_INTERVAL(1000 * 1000)) {
  // do something
}
```

1. `OZ`

用于简化 `OB_FAIL` 之后的日志输出，当在报错后只需要简单输出日志时，可以使用 `OZ`，注意使用 `OZ` 时需要首先在 `.cpp` 文件的开始定义 `USING_LOG_PREFIX`, 例如下面的写法，

```
OZ(func());
```

等价于

```
if (OB_FAIL(func())) {
   LOG_WARN("fail to exec func, ", K(ret));
}
```

1. `K`

通常用于日志输出时，输出变量名与变量值，例如下面的写法，

```
if (OB_FAIL(ret)) {
   LOG_WARN("fail to exec func, ", K(ret));
}
```

1. `KP`

通常用于日志输出时，输出变量名与指针，例如下面的写法，

```
if (OB_FAIL(ret)) {
   LOG_WARN("fail to exec func, ", K(ret), KP(ptr));
}
```

## 继承

所有继承必须是 `public` 的，且使用继承时必须谨慎：只有在“是一个”的情况下使用继承，在“有一个”的情况下使用组合。

当子类继承父类时，子类包含了父类的所有数据及操作定义。C++ 实践中，继承主要用于两种场景：实现继承，子类继承父类的实现代码；接口继承，子类继承父类的方法名称。对于实现继承，由于实现子类的代码在父类和子类间延展，要理解其实现变得更加困难，要谨慎使用。

OceanBase 里面也用到了多重继承，这种场景是很少见的，并且要求最多只有一个基类中包含实现，其他基类都是纯接口类。

## 操作符重载

**除了容器类，自定义数据类型（****`ObString`****、****`ObNumber`** **等）以及** **`ObRowkey`****、****`ObObj`****、****`ObRange`** **等少量全局基础类以外，不要重载操作符（简单的结构的赋值操作除外）**。如果必须违反，请事先征得小组负责人的同意，并详细注释原因。

C++ STL 模板类大量重载操作符，例如，比较函数，四则运算符，自增，自减，这样的代码貌似更加直观，其实往往混淆调用者，例如使得调用者误认为某些比较耗时的操作像内建操作一样高效。

**除了简单的结构外，避免重载赋值符** **`operator =`****。**如果需要的话，可以定义 `deep_copy`、`shallow_copy`等拷贝函数。其中，`deep_copy` 表示所有成员都需要深拷贝，`shallow_copy` 表示其它情况。如果某些成员需要浅拷贝，某些需要深拷贝，那么，采用 `shallow_copy`。

## 声明次序

在头文件中使用特定的声明次序：`public` 块、`protected` 块、`private` 块。

每一块内部的声明次序如下：

1. `typedefs` 和 `enums`
2. 常量
3. 构造函数
4. 析构函数
5. 成员函数，静态成员函数在前，普通成员函数在后
6. 数据成员，静态数据成员在前，普通数据成员在后

宏 `DISALLOW_COPY_AND_ASSIGN` 置于 `private` 块之后，作为类的最后部分。

`.cpp` 文件中的定义次序应该尽可能和 `.h` 中的声明次序保持一致。

之所以要将常量定义放到函数定义（构造/析构函数，成员函数）的前面，而不是放到数据成员中，那是因为常量可能被函数引用。

## 总结

1. 构造函数只做 trival 的初始化工作，每个类都需要定义至少一个构造函数，带有虚函数或者子类的析构函数声明为 `virtual`。

2. 为了避免隐式类型转换，需要将单参数构造函数声明为 `explicit`。

3. **原则上不得使用拷贝构造函数（已经定义使用的基础类除外）。**

   如果必须违反，请事先征得小组负责人的同意，并详细注释原因。

4. 使用 `DISALLOW_COPY_AND_ASSIGN` 避免拷贝构造函数、赋值操作滥用；

5. **类重置使用** **`reset`**，重用使用**`reuse`**，禁止使用 **`clear`**。

6. **需要确保初始化所有成员，且成员变量初始化顺序和定义顺序保持一致。**

7. 仅在只有数据时使用 `struct`，其他情况一概使用 `class`。

8. 每个类包含的通用函数都必须采用标准原型，序列化/反序列化函数必须使用宏实现。

9. 优先考虑组合，只有在“是一个”关系时使用继承。避免私有继承和多重继承，多重继承使用时，要求除一个基类含有实现外，其他基类都是纯接口类。

10. **除了已有的容器类、自定义类型以及少量全局基础类以外，不允许重载操作符（简单的结构的赋值操作除外）**。如果必须违反，请事先征得小组负责人的同意，并详细注释原因。

11. 声明次序：`public` -> `protected` -> `private`。