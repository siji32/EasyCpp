# 引言

本编码规范适用于阿里巴巴/蚂蚁金服 OceanBase 项目，给出了一些编码约束，并定义了编码风格。OceanBase 项目中，测试代码必须遵守本文档的编码风格，建议测试代码也同时遵守本文档的编码约束；其他代码必须遵守本文档的编码约束和编码风格。

本编码规范致力于协助程序员减少编程陷阱，书写出通俗易懂、格式统一的 C/C++ 代码。因此，本规范遵循以下准则：

- 采用最常见、最易懂的方式编写代码。
- 避免采用任何冷僻方式，例如：`foo(int x = 1)。`

- 避免非常技巧的方式，例如：`a+=b; b=a-b; a -= b;` 或者：`a^=b; b^=a; a^=b;` 以交换变量 `a`和 `b` 的值。

本文最后对编码约束进行了小结，以便快速查阅。

本编码规范将根据需要不断进行补充和完善。

# 目录和文件

## 目录结构

OceanBase 系统的子目录说明如下：

- src：存放源代码，包含头文件和实现文件
- unittest：存放单元测试代码和开发人员编写的小规模集成测试代码
- tests：存放测试团队的测试框架和用例
- tools：存放外部工具
- doc：存放文档
- rpm：存放 rpm spec 文件
- script：存放 OceanBase 的运维脚本

C 代码的实现文件命名为 `.c`，头文件命名为 `.h`，C++ 代码的实现文件命名为 `.cpp`，头文件命名为 `.h`。原则上头文件和实现文件必须一一对应，`src` 下的目录和 `unittest` 下的目录一一对应。所有文件命名统一使用全部英文小写字母，单词之间使用’_’分割。

例如，`src` 下的 `common` 目录有一个头文件 `ob_schema.h` 和实现文件 `ob_schema.cpp`，相应地，在 `unittest` 目录下也有一个 `common` 目录，其中有一个名字叫做 `test_schema.cpp` 的单元测试文件。

当然，开发人员也会做一些模块内或者多个模块的集成测试，这些测试代码也放到 `unittest`，但是所在的子目录和文件名不要求和 `src` 一一对应。例如，基线存储引擎的集成测试代码放到 `unittest/storagetest` 目录中。

## 版权信息

目前（2021-3），`observer` & `obproxy` 所有源代码文件头中必须使用如下版权信息：

```
Copyright (c) 2021 OceanBase
OceanBase is licensed under Mulan PubL v2.
You can use this software according to the terms and conditions of the Mulan PubL v2.
You may obtain a copy of Mulan PubL v2 at:
         http://license.coscl.org.cn/MulanPubL-2.0
THIS SOFTWARE IS PROVIDED ON AN "AS IS" BASIS, WITHOUT WARRANTIES OF ANY KIND,
EITHER EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO NON-INFRINGEMENT,
MERCHANTABILITY OR FIT FOR A PARTICULAR PURPOSE.
See the Mulan PubL v2 for more details.
```

## 头文件代码

头文件不能出现实现代码，内联函数或者 C++ 模板除外。另外，如果模板代码太长，可以将其中部分或者全部模板函数提取到专门的 `.ipp` 文件中，具体可参考 `common/ob_vector.h`和 `common/ob_vector.ipp`。

头文件应尽可能简洁清晰，方便使用者理解。如果某些函数的实现代码较短，希望直接放到头文件中，要求将函数声明为内联函数，并且符合内联函数的规范。

## #define 保护

所有头文件都应该使用 `#define` 防止头文件被多重包含，命名格式为：<PROJECT>_<PATH>_<FILE>_。例如：`common` 模块中的头文件 `common/ob_schema.h` 按如下方式保护：

```cpp
#ifndef OCEANBASE_COMMON_OB_SCHEMA_
#define OCEANBASE_COMMON_OB_SCHEMA_
…
#endif // OCEANBASE_COMMON_OB_SCHEMA_
```

## 头文件依赖

在头文件中使用其他类，尽量采用前置声明的方式，避免使用 `#include`。

当一个头文件被包含的同时也引入了新的依赖，一旦该头文件被修改，代码就会被重新编译。如果这个头文件又包含了其他头文件，这些头文件的任何改变都将导致所有包含了该头文件的代码被重新编译。因此，我们倾向于减少包含头文件，尤其是在头文件中包含头文件。

使用前置声明可以显著减少需要包含的头文件数量。举例说明：如果头文件中用到类 `ObFoo` 但不需要访问 `ObFoo` 类的声明，头文件中只需前置声明 `class ObFoo；`而无须 `#include “ob_foo.h”`。

不允许访问类的定义的前提下，我们在一个头文件中能对类 `ObFoo` 做哪些操作？

- 我们可以将数据成员类型声明为 `Ob Foo *`或 `Ob Foo &`.
- 我们可以将函数参数 / 返回值的类型声明为 `ObFoo` (但不能定义实现).

- 我们可以将静态数据成员的类型声明为 `ObFoo`

反之，如果你的类是 `ObFoo` 的子类，或者含有类型为 `ObFoo` 的非静态数据成员，则必须包含 `ob_foo.h` 头文件。

当然，如果使用指针成员代替对象成员会降低代码可读性或者执行效率，那么，不要仅仅为了避免 `#include` 而这么做。

## 内联函数（inline）

为了提高执行效率，有时候我们需要使用内联函数，但需要对内联函数的运行机制有所了解。建议只有在性能关键点，执行代码少于 10 行，不包含循环或者 `switch` 语句，未使用递归机制才使用内联函数。

内联函数在 C++ 类中，应用最广泛的就是用来定义存取函数。一方面，内联该函数可以避免函数调用开销，使得目标代码更加高效；另一方面，每一处内联函数的调用都要复制代码，将使得程序的总代码量膨胀。如果函数体内的代码比较长，或者函数体内出现循环，执行函数体内代码的时间要比函数调用的开销大，则不宜使用内联。

类的构造函数和析构函数容易引起误解。它们可能看起来很短，不过要当心可能隐藏一些行为，如“偷偷地”执行了基类或成员对象的构造和析构函数。

## #include 的路径和顺序

项目内头文件应该按照项目目录树结构引入，不要使用特殊路径，类似 ”.”，”..” 等。为了避免出现多重包含，建议包含头文件的顺序为本文件对应的头文件，系统 C 头文件，系统 C++ 头文件，其它库头文件（Libeasy，tbsys），OceanBase 内部其它头文件。其中，系统 C 头文件用尖括号，末尾加 `.h`，系统C++头文件用尖括号，末尾不加 `.h`，其它情况用引号，例如：

```cpp
#include <stdio.h>，
#include <algorithm>，
#include “common/ob_schema.h”
```

之所以将本文件对应的头文件放到优先位置，是为了减少隐藏依赖。我们希望每一个头文件独立编译，最简单的实现方式是将其作为第一个 `.h` 文件包含在对应的 `.cpp` 文件中。

例如 `ob_schema.cpp` 的 include 包含次序如下：

```cpp
#include "common/ob_schema.h"
 
#include <string.h>
#include <stdlib.h>
#include <assert.h>
 
#include <algorithm>
 
#include "config.h"
#include "tblog.h"
 
#include "common/utility.h"
#include "common/ob_obj_type.h"
#include "common/ob_schema_helper.h"
#include "common/ob_define.h"
#include "common/file_directory_utils.h"
```

## SOURCES 文件

observer 代码顶层目录新增了一个文件 SOURCES，记录只在商业版不能放入开源版的文件路径，请大家自己维护这个文件的内容：

1. 这个文件的格式与 CODEOWNERS 相同
2. 开源版本维护团队将以该文件为准确定哪些文件和目录不包含在开源版之中

## 总结

1. `src` 和 `unittest` 中的子目录一一对应，tests 用来放测试代码。
2. 头文件不能包含实现代码，内联函数和模板除外。
3. 通过 `#define`保护来避免头文件被多重包含。
4. 通过前置声明降低编译依赖，防止修改一个文件引发多米诺效应。
5. 内联函数的合理使用可以提高执行代码效率。
6. 项目内文件的 include 路径为相对路径，include 顺序为：本文件对应的头文件，系统 C 头文件，系统 C++ 头文件，其它库头文件（Libeasy，tbsys），OceanBase 内部其它头文件。

# 命名空间

OceanBase 源代码中的所有变量、函数以及类都通过命名空间区分，命名空间和代码所处的目录一一对应。例如，`src/common` 目录下的 `ob_schema.h` 文件对应的命名空间为 `oceanbase::common`。

```cpp
// .h文件
namespace oceanbase
{
//注意不要缩进
namespace common
{
//所有声明都置于命名空间中，注意不要缩进
class ObSchemaManager
{
public:
 int func();
};
} // namespace common
} // namespace oceanbase
 // .cpp文件
namespace oceanbase
{
namespace common
{
//所有函数实现都置于命名空间中
int ObSchemaManager::func()
{
 …
}
 
} // namespace common
} // namespace oceanbase
```

**危险**



**禁止使用匿名命名空间。**

编译器会给匿名命名空间分配一个随机的命名字符串，这会影响 GDB 调试。

头文件和实现文件中都可能包含对其它命名空间中类的引用。例如，在头文件中声明其它命名空间的类：

```
namespace oceanbase
{
namespace common
{
class ObConfigManager; //类common::ObConfigManager的前置声明
}
 
namespace chunkserver
{
 
class ObChunkServer
{
public:
 int func();
};
 
} // namespace chunkserver
} // namespace oceanbase
```

C++ 允许使用 `using` 的情形如下：

1. `using` 指令（using directive）：例如 `using namespace common`。执行 `using` 指令后，编译器会自动在 common 命名空间中查找符号。
2. `using` 声明（using declaration）：例如 `using common::ObSchemaManager`。执行 `using` 声明后， `ObSchemaManager` 相当于 `common::ObSchemaManager`。

因为 `using` 指令很容易污染作用域，因此，**禁止在** **`.h`** **文件中使用** **`using`** **指令，但允许在** **`.h`** **文件中使用** **`using`** **声明**。`.cpp` 文件中允许使用 `using` 指令，例如，`ObChunkServer` 实现时需要引用 `common` 名字空间的类。注意 `.cpp` 文件只能通过 `using` 指令引入其它命名空间，`.cpp` 文件自身的代码需要放到所在的命名空间中。

例如：

```cpp
//错误的写法
//实现类代码应该放到 chunkserver 名字空间中而不是 using namespace chunkserver;
namespace oceanbase
{
using namespace common;
using namespace chunkserver;
 
//使用 common 命名空间的符号
int ObChunkServer::func()
{
… func函数实现…
}
 
} // namespace oceanbase
 
//正确的写法，实现类代码放到了 chunkserver 命名空间中
namespace oceanbase
{
using namespace common;
 
namespace chunkserver
{
//使用 common 命名空间的符号
int ObChunkServer::func()
{
… func函数实现…
}
 
} // namespace chunkserver
} // namespace oceanbase
```

# 嵌套类

2021-06-16 20:48:33



如果一个类是另外一个类的成员，可以定义为嵌套形式。嵌套类也称为成员类。

```cpp
class ObFoo
{
private:
 // ObBar是嵌套在ObFoo中的成员类/嵌套类，ObFoo称为宿主类/外部类
class ObBar
{
…
};
};
```

当嵌套类只被外部类使用时，将其置于外部类作用域，从而避免污染其他作用域的同名类。另外，建议在外部类的 `.h`文件中前置声明嵌套类，在 `.cpp` 文件中定义嵌套类的实现，这样可以避免外部类的 `.h` 文件中包含嵌套类的实现，提高可读性。

需要注意的是，尽量避免将嵌套类定义为 `public`，除非它们是对外接口的一部分。

# 全局变量与全局函数

2021-06-16 20:48:33



**严格限制全局变量或全局函数的使用**，**除了已有的全局变量和全局函数外，不得增加新的全局变量和全局函数。**如果必须违反，请事先征得小组负责人的同意，并详细注释原因。

全局变量和全局函数带来一系列问题，例如命名冲突，又如全局对象初始化顺序不确定。如果一定要全局共享某个变量，应该放到服务器的单例，例如 `UpdateServer` 的 `ObUpdateServerMain` 中。

全局常量统一定义在 `ob_define.h` 中，全局函数统一定义在 `common/ob_define.h` 和 `utility` 方法（`common/utility.h`，`common/ob_print_utils.h`）中。

**禁止头文件中定义全局** **`const`** **变量**

和“禁止头文件中定义 `static` 变量”原因类似，没有显式 `extern` 的全局 `const` 变量（包括 `constexpr`）也具有 internal linkage，也会在二进制程序中产生多份副本。

**实验分析**

```
// "a.h"const int zzz = 1000;extern const int bbb;
// "a.cpp"#include "a.h"#include <stdio.h>const int bbb = 2000;void func1(){  printf("a.cpp &zzz=%p\n", &zzz);  printf("a.cpp &bbb=%p\n", &bbb);}
// "b.cpp"#include "a.h"#include <stdio.h>void func2(){  printf("b.cpp &zzz=%p\n", &zzz);  printf("b.cpp &bbb=%p\n", &bbb);}
// "main.cpp"void func2();void func1();int main(int argc, char *argv[]){  func1();  func2();  return 0;}
```

编译并执行，可以看到变量 `zzz` 产生了多个实例，变量 `bbb` 只有一个实例。

```
[zhuweng.yzf@OceanBase224004 tmp]$ ./a.outa.cpp &zzz=0x4007e8a.cpp &bbb=0x400798b.cpp &zzz=
```

# 局部变量

2021-06-03 21:26:22



**在语句块开始处（由****{}****组成）声明变量，强制要求简单变量声明时就初始化**。

OceanBase 认为需要在每个语句块（由{}组成）的开始处声明局部变量，这样的代码的可读性较好。另外，允许局部变量在 for 循环语句块的开头声明，如 `for(int i = 0; i < 10; ++i)` 。如果声明变量和使用变量的地方相隔较远，说明语句块包含的代码过多，这往往意味着需要进行代码重构。

在循环体内声明变量，如果变量是一个对象，每次循环都要先后调用其构造函数和析构函数，每次循环也需要圧栈和弹栈，因此将这样的变量提取到循环外要高效得多。**禁止在循环体内声明非简单变量（例如类变量），**如果必须违反，请事先征得小组负责人的同意，并详细注释原因。出于代码可读性的考虑，允许在循环内声明引用。

```cpp
// 低效的实现
for (int i = 0; i < 100000; ++i) {
  ObFoo f;  // 每次进入循环都要调用构造函数和析构函数
  f.do_something();
}
 
// 高效的实现
ObFoo f;
for (int i = 0; i < 100000; ++i) {
  f.do_something();
}
 
//出于代码可读性的考虑，可以在循环内声明引用
for(int i = 0; i < N; ++i) {
   const T &t = very_long_variable_name.at(i);
   t.f1();
   t.f2();
   ...
}
```

此外，OceanBase 对局部变量的大小进行限制，不建议定义过大的局部变量。

1. 函数栈不超过 32K。
2. 单个局部变量不超过 8K。



# 静态变量

2021-06-16 20:48:34



**禁止头文件中定义静态变量**

除了下述**【例外】**所描述的情况外，不允许在 `.h` 头文件中定义初始化静态变量（**无论是不是 const 常量**）。否则，这样的静态变量在每个编译单元（`.o` 文件）中会产生一个静态区存储的变量，链接后会有多个静态变量实例。如果该变量是 const 变量，链接后将造成编译后二进制程序文件膨胀。如果不是 const 变量，则可能造成严重的 bug。



注意，是禁止定义（define），不是禁止声明（declare）。

**【例外】static const/constexpr 静态成员变量**

类里的 `static const int`（包括 `int32_t`, `int64_t`, `uint32_t`, `uint64_t` 等），`static constexpr double` 等静态成员变量，我们常用来定义 hardcode 数组长度等，是不占存储的，没有地址（可以视为 `#define` 宏常量），允许在头文件中定义初始化。



也就是说，下面这种形式（伪代码）是允许的：

```
class Foo {
  static const/constexpr xxx = yyy;
};
```

对这条例外的解释如下。在 C++98 中，允许 `static const int`变量在声明时定义值。

```
class ObBar
{
public:
  static const int CONST_V = 1;
};
```

事实上，C++ 编译器认为上面这段代码等价于：

```
class ObBar
{
  enum { CONST_V = 1 };
}
```

如果在程序中取这种变量的地址，链接的时候会产生 "Undefined reference" 错误。对于这种情况，正确的做法是把变量的定义放入 `.cpp` 中。

```
// 在.h中
class ObBar
{
      static const int CONST_V;
}
// 在.cpp中：
const int ObBar::CONST_V = 1;
```

在 C++11 之前，C++98 标准只允许整型类型的 `static const` 变量在类声明中包含定义进行初始化。在C++11 中，引入了 `constexpr`，用 `static constexpr` 成员变量（包括 `double` 等类型）也可以在声明中进行初始化。这种变量在编译后也不会产生静态区存储。

```
Before C++11, the values of variables could be used in constant expressions only if the variables are declared const, have an initializer which is a constant expression, and are of integral or enumeration type. C++11 removes the restriction that the variables must be of integral or enumeration type if they are defined with the constexpr keyword:

constexpr double earth_gravitational_acceleration = 9.8;
constexpr double moon_gravitational_acceleration = earth_gravitational_acceleration / 6.0;

Such data variables are implicitly const, and must have an initializer which must be a constant expression.
```

**案例一**

按照现在 OceanBase 的代码风格，静态变量被放在头文件中定义（例如 `ob_define.h`)，每个 `.cpp` 文件 include 这类头文件的时候都会生成一次这个变量的声明和定义。如果该静态变量是 latch，wait event 等大对象，那么将导致生成的二进制文件大小和内存的占用剧烈膨胀。然而，只要简单的将几个静态变量的定义由头文件移至 `.cpp` 文件，并把头文件中的静态变量改成 `extern` 定义，就可以显著改善这一问题。

**案例二**

下面这个例子中，不同的 `.cpp` 看到的是全局变量的不同副本。本来预期是通过全局的静态变量来通信，结果变成了各说各话。这样还会造成“假”的单例实现。

**静态变量行为分析**

我们写一个小程序来验证一下静态变量定义在 `.h` 中的表现。

```
// "a.h"
static unsigned char xxx[256]=
{
  1, 2, 3
};
static unsigned char yyy = 10;
static const unsigned char ccc = 100;
// "a.cpp"
#include "a.h"
#include <stdio.h>
void func1()
{
  printf("a.cpp &xxx=%p\n", xxx);
  printf("a.cpp &yyy=%p\n", &yyy);
  printf("a.cpp &ccc=%p\n", &ccc);
}
// "b.cpp"
#include "a.h"
#include <stdio.h>
void func2()
{
  printf("b.cpp xxx=%p\n", xxx);
  printf("b.cpp &yyy=%p\n", &yyy);
  printf("b.cpp &ccc=%p\n", &ccc);
}
// "main.cpp"
void func2();
void func1();
int main(int argc, char *argv[])
{
  func1();
  func2();
  return 0;
}
```

编译并执行，可以看到无论是静态整数还是数组，无论是否是 const 变量，都产生了多个实例。

```
[zhuweng.yzf@OceanBase224004 tmp]$ g++ a.cpp b.cpp main.cpp
[zhuweng.yzf@OceanBase224004 tmp]$ ./a.out
a.cpp &xxx=0x601060
a.cpp &yyy=0x601160
a.cpp &ccc=0x400775
b.cpp xxx=0x601180
b.cpp &yyy=0x601280
b.cpp &ccc=0x4007a2
```

# 资源回收与参数恢复

2021-06-16 20:48:35



**资源管理遵守“谁申请谁释放”的原则，并在语句块结束时统一释放资源**。如果必须违反，请事先征得小组负责人的同意，并详细注释原因。

每个语句块的代码结构如下：

1. 变量定义
2. 资源申请
3. 业务逻辑
4. 资源释放

```cpp
// 错误的用法void *ptr = ob_malloc(sizeof(ObStruct), ObModIds::OB_COMMON_ARRAY);
if (NULL == ptr) {
 // print error log
} else {
 if (OB_SUCCESS != (ret = do_something1(ptr))) {
// print error log
ob_tc_free(ptr, ObModIds::OB_COMMON_ARRAY);
ptr = NULL;
 } else if (OB_SUCCESS != (ret = do_something2(ptr))) {
// print error log
ob_free(ptr, ObModIds::OB_COMMON_ARRAY);
ptr = NULL;
 } else { }
}
 
//正确的用法
void *ptr = ob_malloc(100, ObModIds::OB_COMMON_ARRAY);
if (NULL == ptr) {
 // print error log
} else {
 if (OB_SUCCESS != (ret = do_something1(ptr))) {
// print error log
 } else if (OB_SUCCESS != (ret = do_something2(ptr))) {
// print error log
 } else { }
}
//释放资源
if (NULL != ptr) {
ob_free(ptr, ObModIds::OB_COMMON_ARRAY);
ptr = NULL;
}
```

上面的例子中，最外层的 if 分支只是判断资源申请失败的情况，由 else 分支处理业务逻辑。因此，也可以将资源释放的代码放在最外层 else 分支的末尾。

```cpp
//另外一种正确的写法，要求if分支只是简单处理资源申请失败
void *ptr = ob_malloc(100, ObModIds::OB_COMMON_ARRAY);
if (NULL == ptr) {
 // print error log
} else {
 if (OB_SUCCESS != (ret = do_something1(ptr))) {
// print error log
 } else if (OB_SUCCESS != (ret = do_something2(ptr))) {
// print error log
 } else { }
//释放资源
ob_free(ptr, ObModIds::OB_COMMON_ARRAY);
ptr = NULL;
}
```

因此，如果需要释放资源，在函数返回前或者最外层 else 分支的末尾，统一释放资源。

某些情况下需要在语句块的开头保存输入参数，并在异常时恢复参数。和资源回收类似，只能在语句块的结尾恢复参数。最典型的例子是序列化函数，例如：

```cpp
//错误的写法
int serialize(char *buf, const int64_t buf_len, int64_t &pos)
{
 int ret = OB_SUCCESS;
 const int64_t ori_pos = pos;
 
 if (OB_SUCCESS != (ret = serialize_one(buf, buf_len, pos)) {
pos = ori_pos;
…
 } else if (OB_SUCCESS != (ret = serialize_two(buf, buf_len, pos)) {
pos = ori_pos;
…
 } else {
   …
 }
 return ret;
}
```

这种用法的问题在于很可能会在某个分支忘记恢复 `pos` 的值，正确的写法如下：

```cpp
//正确的写法
int serialize(char *buf, const int64_t buf_len, int64_t &pos)
{
 int ret = OB_SUCCESS;
 const int64_t ori_pos = pos;
 
 if (OB_SUCCESS != (ret = serialize_one(buf, buf_len, pos)) {
…
 } else if (OB_SUCCESS != (ret = serialize_two(buf, buf_len, pos)) {
…
 } else {
   …
 }
 
 if (OB_SUCCESS != ret) {
pos = ori_pos;
 }
 return ret;
}
```

因此如果需要恢复输入参数，在函数返回前恢复。



# 总结

2021-06-16 20:48:35



1. 命名空间和目录对应，**禁止使用匿名命名空间**，**`.h`** **文件中禁止使用** `using` **指令，只允许使用** **`using`** **声明**。
2. 嵌套类适合用在只被外部类使用的场景，建议在 `.h` 文件中前置申明，在 `.cpp` 文件中实现，尽量不要用 `public`。
3. **除了已有的全局变量和全局函数外，不得增加新的全局变量和全局函数。**如果必须违反，请事先征得小组负责人的同意，并详细注释原因。
4. **局部变量在语句块开头处声明，强制要求简单变量声明时就初始化。**
5. **禁止在循环体内声明非简单变量。**如果必须违反，请事先征得小组负责人的同意，并详细注释原因。
6. **资源管理遵守“谁申请谁释放”的原则。**如果需要释放资源，在函数返回前或者最外层 else 分支的末尾释放。因此如果需要恢复输入参数，在函数返回前恢复。如果必须违反，请事先征得小组负责人的同意，并详细注释原因。



# 单入口单出口

2021-06-16 20:48:43



强制要求所有函数在末尾返回，禁止中途调用 `return`、`goto`、`exit` 等全局跳转指令。如果必须违反，请事先征得项目负责人和项目架构师的同意，并详细注释原因。

OceanBase 认为大规模项目开发应该优先避免常见陷阱，牺牲编程复杂度是值得的。而单入口单出口能够使得开发人员不容易忘记释放资源或者恢复函数输入参数。无论任何时候，都要求函数只有一个出口。

# 函数返回值

2021-06-16 20:48:44



**除了如下几种例外，函数都必须返回****ret****错误码**：

1. 简单存取函数 `set_xxx()`/`get_xxx()`。如果 `set`/`get` 函数比较复杂或者可能出错，那么仍然必须返回错误码。
2. 已经定义和使用的 `at(i)` 函数等（定义和使用新的，请事先征得小组负责人的同意，并详细注释原因）。
3. 已经定义和使用的操作符重载（定义和使用新的，请事先征得小组负责人的同意，并详细注释原因）。
4. 其他少量函数，例如类的通用函数 `void reset();` 、`void reuse();`等参见 4.7 节通用函数。

**函数调用者必须检查函数的返回值（错误码）并处理。**

**只能用** `int` **类型的** `ret` **变量表示错误，且** `ret` **只能表示错误****(****迭代器函数由于历史原因除外****)**。如果需要返回其它类型的值，比如 `compare` 函数返回 `bool` 类型的值，需要采用其它变量名，例如 `bool_ret`。例如：

```cpp
//错误的写法
bool operator()(const RowRun &r1, const RowRun &r2) const
{
 bool ret = false;
 
 int err = do_something();
 
 return ret;
}
 
//正确的写法
bool operator()(const RowRun &r1, const RowRun &r2) const
{
 bool bool_ret = false;
 
 int ret = do_something();
 
 return bool_ret;
}
```

如果函数执行过程中需要临时保存一些错误码，那么，尽量使用含义明确的变量，例如 `hash_ret`，`alloc_ret`。如果含义不明确，那么，也可以依次采用 `ret1`，`ret2`，…… , `retn` 来表示，避免使用 `err` 表示错误码引起混淆。例如：

```cpp
int func()
{
 int ret = OB_SUCCESS;
 
 ret = do_something();
 if (OB_SUCCESS != ret) {
  int alloc_ret = clear_some_resource ();
  if (OB_SUCCESS != alloc_ret) {
// print error log
  }
 } else {
   …
 }
 return ret;
}
```



# 顺序语句

2021-06-16 20:48:44



如果多条顺序语句在做同一件事情，那么，在某些情况下可以采用精简写法。

由于函数执行过程中需要判断错误，这将使得顺序代码变得冗长。例如：

```cpp
//冗长的代码
int ret = OB_SUCCESS;
 
ret = do_something1();
if (OB_SUCCESS != ret) {
 // print error log
}
 
if (OB_SUCCESS == ret) {
 ret = do_something2();
 if (OB_SUCCESS != ret) {
// print error log
 }
}
//更多代码。。。
```

可以看出，真正有效的代码只有两行，但是整体代码量是有效代码的好几倍。这将使得一屏包含的有效代码过少，影响可读性。

如果顺序语句中每一步都只需要一行代码，那么，建议通过如下的方式精简代码：

```cpp
//当顺序逻辑中每一步只有一行代码时，使用精简写法
int ret = OB_SUCCESS;
 
if (OB_FAIL(do_something1())) {
// print error log
} else if (OB_FAIL(do_something2())) {
 // print error log
} else if (OB_SUCCESS != (ret = do_something3())) {
 // print error log
} else { }
```

如果顺序语句的某些步骤超过一行代码，那么，需要做一些变化：

```cpp
//当顺序逻辑中某些步骤超过一行代码，使用精简写法，并做一定变化
int ret = OB_SUCCESS;
 
if (OB_SUCCESS != (ret = do_something1())) {
// print error log
} else if (OB_SUCCESS != (ret = do_something2())) {
 // print error log
} else {
 //步骤3执行的代码超过一行
 if (OB_SUCCESS != (ret = do_something3_1())) {
// print error log
 } else if (OB_SUCCESS != (ret = do_something3_2()) {
// print error log
} else { }
}
 
if (OB_SUCCESS == ret) { //开始一段新的逻辑
 if (OB_SUCCESS != (ret = do_something4())) {
// print error log
 } else if (OB_SUCCESS != (ret = do_something5())) {
// print error log
 } else { }
}
```

实际编码过程中，什么时候应该采用精简写法呢？OceanBase 认为，当顺序语句的每一步都只有一行语句，并且这些步骤逻辑上耦合比较紧，都应该尽量采用精简写法。然而，如果逻辑上属于多个部分，每个部分做不同的事情，那么，只应该在每个部分内部采用精简写法，而不是为了精简而精简。

需要注意的是，如果顺序语句后面紧接着条件语句。假如顺序语句采用精简写法变成条件语句，那么，不能将它们合并为一个大的条件语句，而应该将它们在代码结构上分开来。例如：

```cpp
//错误的写法
if (OB_SUCCESS != (ret = do_something1())) {
// print error log
} else if (OB_SUCCESS != (ret = do_something2())) {
 // print error log
} else if (cond) {
 // do something if cond
} else {
 // do something if !cond
}
 
//第一种正确的写法
if (OB_SUCCESS != (ret = do_something1())) {
// print error log
} else if (OB_SUCCESS != (ret = do_something2())) {
 // print error log
} else {
if (cond) {
   // do something if cond
} else {
   // do something if !cond
}
}
 
//第二种正确的写法
if (OB_SUCCESS != (ret = do_something1())) {
// print error log
} else if (OB_SUCCESS != (ret = do_something2())) {
 // print error log
} else { }

if (OB_SUCCESS == ret) {
if (cond) {
   // do something if cond
} else {
   // do something if !cond
}
}
```



# 循环语句

2021-06-16 20:48:45



在循环条件中判断 `OB_SUCCESS == ret`，防止错误码被覆盖等问题。

OceanBase 发现了大量错误码被覆盖的问题，这些问题往往都会导致严重后果，例如数据不一致，而且非常难以发现。例如：

```cpp
//错误码被覆盖
for (int i = 0; i < 100; ++i) {
 ret = do_something();
 if (OB_SUCCESS != ret) {
// print error log
 } else {
…
 }
}
```

上面的例子中，`for` 循环中的 `if` 分支判断错误，但是忘记 `break`，这样，代码将会进入下一次循环，前一次执行的错误码将被覆盖。

因此，规范 `for` 循环语句的写法如下：

```cpp
for (int i = 0; OB_SUCCESS == ret && i < xxx; ++i) {
 …
}
```

另外，规范 `while` 循环语句的写法如下：

```cpp
while (OB_SUCCESS == ret && other_cond) {
 …
}
```

循环语句中可能会用到 `break` 或者 `continue` 来改变执行路径。OceanBase 认为应该尽量少用，这和函数单入口单出口的道理是一样的，相当于循环语句的后续代码的输入来源为多个入口，增加了代码的复杂度。如果确实有必要使用 `break` 和 `continue`，要求通过注释详细说明原因，写代码或者 `Review` 代码时都需要格外关注。另外，考虑到后续代码的输入来源为多个入口，需要确保能说清楚后续代码的输入到底满足什么条件。



# 条件语句

2021-06-16 20:48:45



条件语句需要遵守 MECE 原则。

MECE 一词来源于麦肯锡分析法，意思是相互独立，完全穷尽（Mutually Exclusive Collectively Exhaustive）。原则上，每个条件语句的 `if/else` 分支都需要完全穷尽所有可能。

一些不好的编程风格，例如：

```cpp
//不好的编程风格
if (OB_SUCCESS != ret && x > 0) {
 // do something
}
 
if (OB_SUCCESS == ret && x < 0 ) {
 // do something
}
 
if (OB_SUCCESS == ret && x == 0) {
 // do something
}
```

这样的代码不符合 MECE 原则，很难分析是否穷尽了所有的可能，很容易遗漏一些场景。

如果只有一个条件，正确的写法是：

```cpp
//一个判断条件的正确写法
if (cond) {
 // do something
} else {
 // do something
}
```

原则上，每个 `if/else` 分支都是完整的，即使最后一个 `else` 分支什么也不做。不过有一种情况例外，如果 `if` 条件只是一些错误判断或者参数检查，没有其它逻辑，那么，`else` 分支可以省略。

```cpp
// if语句只是判断错误码，else分支可省略
if (OB_SUCCESS != ret) {
 //处理错误
}
如果包含两个判断条件，对比如下两种可能的写法：
//两个判断条件的第一种写法（正确）
if (cond1) {
 if (cond2) {
   // do something
 } else {
// do something
 }
} else {
 if (cond2) {
// do something
 } else {
// do something
 }
}
 
//两个判断条件的第二种写法（错误）
if (cond1 && cond2) {
 // do something
} else if (cond1 && !cond2) {
 // do something
} else if (!cond1 && cond2) {
 // do something
} else {
 // do something
}
```

第一种写法分为两层，第二种写法分为一层，OceanBase 只允许第一种写法。当然，这里的 cond1 和 cond2 是从业务逻辑的角度说的，指的是两段独立的业务逻辑，而不是说 cond1 和 cond2 里面不能包含 `&&` 或者 `||` 运算符。例如：

```cpp
// app_name 是否为空，包含 ||
if (NULL == app_name || app_name[0] == ‘\0’) {
 …
}
 
//判断 table_name 或者 column_name 是否为空，认为是一个业务逻辑
if (NULL != table_name || NULL != column_name) {
 …
}
```



无论如何，每一层的 `if/else` 分支个数不超过 5 个。为什么选择 5 个？这个数字也来自麦肯锡分析法，一般来讲，同一层次的分支逻辑一般在 3~5 个之间，如果超过了，往往是划分不合理。

# const 声明

2021-06-03 20:02:47



将不会发生变化的函数参数声明为 const。另外，如果函数不修改成员变量，也应该声明为 const 函数。

将参数声明为 const 可以避免一些不必要的错误，例如不变的参数因为代码错误被改变了。对于简单数据类型值传递，很多人对是否声明为 const 存在争议，因为这种情况声明 const 没有任何效果。

考虑到 OceanBase 已有代码大多都已经声明为 const，而且这样操作起来要更加容易，因此，只要函数参数不会发生变化，统一声明为 const。

# 函数参数

2021-06-16 20:48:46



函数参数不得超过 7 个，建议的顺序为：输入参数在前，输出参数在后，如果某些参数既是输入参数又是输出参数，当成输入参数处理，和其他输入参数一样放在前面，添加新的参数也需要遵守这个原则。

**编码的原则：代码上不相信任何人！每个函数（无论** **`public`** **还是** **`private`****，内联函数除外）必须检查每个输入参数的合法性，强烈建议内联****都必须检查从类成员变量或者通过函数调用获得的值（例如** **`get`** **返回值或输出参数）的合法性，即使返回值为成功，也仍然要检查输出参数的合法性。变量（参数）检查，一个函数内只需要检查一次（如果多次调用一个或几个函数获得的值，那么每次都要检查）。**这些检查包括但不限于：

1. 指针是否为 `NULL`，字符串是否为空
2. 数值类型参数值是否超过值域，特别地，数组/字符串/缓冲区的下标是否越界
3. 对象类型的参数是否有效，一般地，对象可以定义一个 `bool is_valid()` 方法（参考 `common::TableSchema`）

如果在函数内已经做了隐式的检查，例如通过一个检查函数，那么要在变量赋值的地方予以说明。例如：

```cpp
//隐式检查过的变量，要在变量赋值的地方予以说明：
if (!param.is_valid() || !context.is_valid()) {
   ret = OB_INVALID_ARGUMENT;
   STORAGE_LOG(WARN, "Invalid argument", K(ret), K(param), K(param));
 } else {
   // block_cache_非空已在前面的context.is_valid()中检查过
   ObMicroBlockCache *block_cache = context.cache_context_.block_cache_;
...
}
```

使用 `if` 语句检查输入参数（函数本身）和输出参数（函数调用者）的合法性，**任何时候都禁止使用** **`assert`****、禁止使用先前定义的** **`OB_ASSERT`** **宏**。

示例如下：

```cpp
//需要返回错误的函数
int _func(void *ptr)
{
 int ret = OB_SUCCESS;
 
 if (NULL == ptr) {
// print error log
ret = OB_INVALID_ARGUMENT;
 }
 else {
  //执行业务逻辑
 }
 return ret;
}
```

# 函数调用

2021-06-16 20:48:46



函数调用时应该尽量避免传入一些无意义的特殊值，例如 `NULL`、`true/false`、`0/-1` 等，而应该使用常量来替代。如果一定要传入特殊值，需要采用注释说明。

例如：

```cpp
//错误的写法
int ret = do_something(param1, 100, NULL);
 
//正确的写法
ObCallback *null_callback = NULL;
int ret = do_something(param1, NUM_TIMES, null_callback);
```

# 指针和引用

2021-06-16 20:48:47



函数参数可以选择指针，也可以选择引用。在遵守惯用法的前提下，更多地使用引用。

指针参数和引用参数往往可以达到同样的效果。考虑到 OceanBase 编码规范中对错误判断要求比较严格，因此，更多地使用引用，减少一些冗余的错误判断代码。当然，前提是必须遵守惯用法，例如：

1. 申请对象的方法返回往往一个指针，相应的释放方法传入的也是指针。
2. 如果对象的成员是一个指针，相应的 `set_xxx` 传入的也是指针。

# 函数长度

2021-06-03 20:45:42



强制要求单个函数不超过 120 行。如果必须违反，请事先征得小组负责人的同意，并详细注释原因。

大多数开源项目都会限制单个函数的行数，一般来讲，80 行以上的函数往往都是不合适的。考虑到 OceanBase 有大量冗余的错误判断代码，限制单个函数不超过 120 行。如果函数过长，考虑将其分割为更加短小、易于管理的若干个函数，或者重新审视设计，修改类的结构。

# 总结

2021-06-03 20:54:27



1. **严格遵守函数单入口单出口。如果必须违反，请事先征得项目负责人和项目架构师的同意，并详细注释原因**

   。

2. **除了简单存取函数** **`set_xxx()`****/****`get_xxx()`****``****和少量例外(如操作符重载，已有的** **`at(i)`** **函数，类的通用函数** **`reset()`****/****`reuse()`****等)，所有函数（****`public`** **和** **`private`****）都应该用** **`ret`** **返回错误码，如果** **`set/get`** **比较复杂或可能出错，仍然要用** **`ret`** **返回错误码。**

   只能用 `int` 类型的 `ret` 变量表示错误，且 `ret` 只能表示错误(迭代器函数由于历史原因除外)。

3. 如果多条顺序语句在做同一件事情，那么，在某些情况下可以采用精简写法。

4. 在循环条件中判断 `OB_SUCCESS == ret`，防止错误码被覆盖等问题。

5. 条件语句需要遵守 MECE 原则：各个条件之间相互独立，完全穷尽，且单个 if/else 的分支个数尽量不超过5个。

6. 尽可能将函数/函数参数声明为 `const`。

7. **编码的原则：代码上不相信任何人！每个函数（无论** **`public`** **还是** **`private`****，内联函数除外）必须检查每个输入参数的合法性，强烈建议内联函数也进行这些检查（除非有严重性能问题）。所有函数（无论** **`public`** **还是** **`private`****）都必须检查从类成员变量或者通过函数调用获得的值（例如 get 返回值或输出参数）的合法性，即使返回值为成功，也仍然要检查输出参数的合法性。变量（参数）检查，一个函数内只需要检查一次（如果多次调用一个或几个函数获得的值，那么每次都要检查）。**

   定义函数时，建议的顺序为：输入参数在前，输出参数在后。

8. **禁止使用** **`assert`** **和** **`OB_ASSERT`****。**

9. 函数调用时应该尽量避免传入一些无意义的特殊值，而采用常量替代。

10. 在遵守惯用法的前提下，更多地使用引用。

11. 强制要求单个函数不超过 120 行。如果必须违反，请事先征得小组负责人的同意，并详细注释原因。

# 智能指针与资源 Guard

2021-06-16 20:48:48



不允许使用智能指针，允许通过 `Guard` 类自动释放资源。

boost 库支持智能指针，包括 `scoped_ptr`、`shared_ptr` 以及 `auto_ptr`。很多人认为智能指针能够被安全使用，尤其是 `scoped_ptr`，不过 OceanBase 已有代码大多都手动释放资源，且智能指针用得不好容易有副作用，因此，不允许使用智能指针。

允许用户手写一些 `Guard` 类，这些类的方法会申请一些资源，这些资源会在类析构的时候自动释放掉，例如 `LockGuard` 和 `SessionGuard`。

# 内存申请与释放

2021-06-16 20:48:48



要求使用内存分配器申请内存，内存释放后要立即将指针置为 `NULL`。

OceanBase 可用于内存分配的方法包括 `ob_malloc` 以及各种内存分配器，要求使用内存分配器申请内存，且申请时指定所属的模块。这样的好处是方便系统管理内存，如果出现内存泄露，很容易看出是哪个模块。另外，需要防止引用已经释放的内存空间，要求在调用 `free` 方法之后立刻将指针置为 `NULL`。

```cpp
void *ptr = ob_malloc(100, ObModIds::OB_COMMON_ARRAY);
 
// do something
 
if (NULL != ptr) {
//释放资源
ob_free(ptr, ObModIds::OB_COMMON_ARRAY);
ptr = NULL; // free之后立即将指针置空
}
```

# 字符串

2021-06-16 20:48:49



禁止使用 `std::string` 类，而是采用 `ObString` 代替。另外，操作 C 字符串时，要求使用限长字符串函数。

虽然 C++ 的 `std::string` 类在使用时非常方便，但是程序员无法时刻确保自己明确其内部的具体行为（例如拷贝，隐式转换等）。因此 OceanBase 要求尽量使用 `ObString`，其中用到的内存需要开发者手动管理。

如果要使用 C 字符串，要使用限长字符串操作函数：`strncat`、`strndup`、`snprintf`、`memcpy`，不要使用不限制长度的字符串操作函数：`strcpy`、`strcat`、`strdup`、`sprintf` 和 `strncpy`。允许使用 `strlen` 来获取字符串的长度。禁止使用 `strncpy` 的原因是该函数存在性能问题，并且，如果 `src` 参数的长度小于参数 `n` 的时候，该函数不会在末尾自动补 `’\0’` 。可使用 `memcpy`、`snprintf` 来替换 `strncpy`。

# 数组/字符串/缓冲区访问

2021-05-31 23:09:22



函数把数组/字符串/缓冲区作为参数传递时，必须同时传递数组/字符串/缓冲区的长度。访问数组/字符串/缓冲区的内容时，必须检查下标是否越界。

# 友元

2021-06-16 20:48:49



只能在同一个文件中使用友元，如果必须违反，请事先征得小组负责人的同意，并详细注释原因。将单元测试类声明为友元可以例外，但需谨慎使用。

通常将友元定义在同一个文件中，避免代码阅读者跑到其它文件中查找其对某个类私有成员的使用。经常用到友元的场景包括：

1. 迭代器：往往会将迭代器类声明为友元，例如 `ObQueryEngine` 将 `ObQueryEngineIterator` 声明为友元。
2. 工厂模式：例如将 `ObFooBuilder` 声明为 `ObFoo` 的友元，以便 `ObFooBuilder` 访问 `ObFoo` 的内部状态。

某些情况下，为了提高测试覆盖率，将一个单元测试类声明为待测类的友元会很方便。不过，需要谨慎对待这种做法。大部分情况下，我们应该通过 `public` 函数的各种输入组合间接地测试 `private` 函数，否则，这些单元测试代码将会难以维护。

# 异常

2021-06-16 20:48:50



禁止使用 C++ 的异常机制。

虽然包括 Java 在内的部分编程语言鼓励使用异常机制，但是，这会使得程序控制流变得更加复杂，再加上程序员可能忘记捕获某些异常，反而为后续调试和改 bug 带来不便，因此，禁止使用异常机制，而是采用错误码来返回错误。

# 运行时类型识别

2021-06-16 20:48:50



禁止使用运行时类型识别（RTTI）。

程序在运行时对类型进行识别往往意味着程序设计存在问题，如果一定要使用 RTTI，则有必要重新考虑类的设计。

# 类型转换

2021-06-16 20:48:51



使用 `static_cast<>` 等 C++ 类型转换，禁止使用 C 风格的强制类型转换，如 `int y = (int) x`。

C++ 风格的类型转换包括：

1. `static_cast`：和 C 风格相似可以做值的强制转换，或者指针的子类到父类的明确的向上转换。
2. `const_cast`：移除 const 属性。
3. `reinterpret_cast`：指针类型和整数或其它指针间不安全的相互转化，使用时需要谨慎。
4. `dynamic_cast`：除了测试代码，禁止使用。

# 输出

2021-06-16 20:48:52



尽量采用 `to_cstring` 输出。

原则上，每个支持打印的类都需要实现 `to_string`，

`to_cstring` 使用示例如下：

```cpp
FILL_TRACE_LOG("cur_trans_id=%s", to_cstring(my_session->get_trans_id()));
FILL_TRACE_LOG("session_trans_id=%s", to_cstring(physical_plan->get_trans_id()));
```

# 整型

2021-06-16 20:48:52



返回的错误码使用 `int` 类型，函数参数和循环次数变量尽量使用 `int64_t` 类型。其它情况使用指定长度的有符号数，例如 `int32_t`，`int64_t`。避免使用无符号数，少数惯用法除外。

函数参数和循环次数之所以尽量使用 `int64_t`，是为了避免函数调用和循环语句中出现大量的数据类型转化。当然，惯用法可以除外，例如端口为 `int32_t`。而在 `struct` 这样的结构体中，往往会有 8 字节对齐或者节省内存的需求，因此，可以使用指定长度的有符号数。

除了 bit set 或者编号（例如 table_id）这样的惯用法以外，都应该避免使用无符号数。无符号数可能带来一些隐患，例如：

```cpp
for (unsigned int i = foo.length() – 1; i >= 0; --i)
```

上述代码永远不会终止。

对于编号，目前的代码有的采用 `0` 作为非法值，有的采用 `MAX_UINT64` 作为非法值，以后统一规定：`0` 以及`MAX_UINT64` 都是非法值，且在 utility 中提供内联函数 `is_valid_id` 用于检查。另外，统一将编号值初始化为宏 `OB_INVALID_ID`，宏 `OB_INVALID_ID` 的初始值调整为 `0`。

# sizeof

2021-06-16 20:48:53



尽量使用 `sizeof(var_name)` 代替 `sizeof(type)`。

这是因为，如果 `var_name` 的类型发生变化，`sizeof(var_name)` 会自动同步，而 `sizeof(type)` 不会，这可能会带来一些隐患。

```cpp
ObStruct data;
memset(&data, 0, sizeof(data)); //正确的写法
memset(&data, 0, sizeof(ObStruct)); //错误的写法
```



需要注意的是，不要使用 `sizeof` 计算字符串的长度，而改用 `strlen`。例如：

```cpp
Char *p = “abcdefg”;
int64_t nsize = sizeof(p); // sizeof(p)表示指针大小，在64位机上等于8
```

# 0 与 nullptr

2021-05-31 23:09:27



整数用0，实数用0.0，指针用nullptr（替代之前的NULL），字符串用’\0’。

# 预处理宏

2021-06-16 20:48:53



**除了已有的宏之外，不得定义新的宏，而是以内联函数、枚举或常量代替。**如果必须定义新的宏，请事先征得小组负责人的同意，并详细注释原因。

宏可以做一些其它技术无法实现的事情，例如字符串化（stringifying，使用#），连接（concatenation，使用##）。宏往往用于输出和序列化，例如 `common/ob_print_utils.h`，`rpc 相关类`，然而，很多时候，可以使用其它方式代替宏：宏内联效率关键代码可以通过内联函数替代，宏存储常量可以使用 const 变量替代。

判断的原则是：除了输出和序列化，只要能不用宏，尽量不要用宏。

# Boost 与 STL

2021-05-31 23:09:28



**STL****中只允许使用****<algorithm>****头文件定义的算法类函数，如****std_sort****，禁止使用其它****STL****或****boost****功能**。**如果必须违反，请事先征得项目负责人和项目架构师的同意，并详细注释原因。**

OceanBase对boost和STL这样的库持保守态度，我们认为正确地编写代码的重要性远远高于方便地编写代码。除了STL <algorithm>定义的算法类函数，其它功能都不应该使用。

 

# auto

2021-06-16 20:48:54



`auto` 机制允许程序员在声明变量的时候免去写出变量的具体类型，而是由编译器根据初始化表达式自动推导类型。例如：

```cpp
auto i = 42;        // i is an intauto l = 42LL;      // l is an long longauto p = new foo(); // p is a foo*
```

**危险**



OceanBase 禁止使用 auto 机制

禁止原因：虽然 `auto` 可以使得一些模板类型的声明更短，但是我们希望类型的声明符合使用者的意图，因此禁止在代码中使用 `auto` 机制。

# Range-based for loops

2021-06-16 20:48:55



`Range-based for loops` 是新的 `for` 循环语法，用来方便地对提供 `begin()` 和 `end()` 的容器进行遍历。例如：

```cpp
for(const auto& kvp : map) {
  std::cout << kvp.first << std::endl;
}
```

**是否允许**

**危险**



OceanBase 禁止使用 Range-based for loops

禁用 `Range-based for loops` 的原因：该特性只是一个语法糖。之前 OceanBase 的代码中已经大量使用了自己定义的 `FOREACH` 宏，可以达到类似的效果。

# Override and final

2021-06-16 20:48:55



`override` 用来表明一个虚函数是对基类中虚函数的重载；`final`表明某个虚函数不能被派生类重载。例如：

```cpp
class B{public:virtual void f(short) {std::cout << "B::f" << std::endl;}virtual void f2(short) override final {std::cout << "B::f2" << std::endl;}};
class D : public B{public:virtual void f(int) override {std::cout << "D::f" << std::endl;}};
class F : public B{public:virtual void f2(int) override {std::cout << "D::f2" << std::endl;} // compiler error};
```

**说明**



**OceanBase 允许使用 override 和 final**

允许原因：根据之前的经验，OceanBase 中虚函数重载漏了 `const` 导致重载错误的错误层出不穷，该特性有助于减少该类错误的发生。

# strongly-typed enums

2021-06-16 20:48:56



相较于传统的枚举类型，Strongly-typed enums 不会将枚举元素暴露到外层作用域中，也不会隐式转换为整形，并且可以拥有用户指定的元素类型。例如

```cpp
enum class Options {None, One, All};
Options o = Options::All;
```

**说明**



**OceanBase 允许使用** Strongly-typed enums

允许原因：新的枚举类型让编译器的检查更加严格，且使用新的关键字定义，和原来的 enum 不冲突。

# Lambdas

2021-06-16 20:48:56



Lambdas 是从函数式编程中借鉴的概念，用来方便地书写匿名函数。例如：

```
std::function<int(int)> lfib = [&lfib](int n) {return n < 2 ? 1 : lfib(n-1) + lfib(n-2);};
```

**危险**



OceanBase 禁止使用 Lambdas。

禁用 Lambdas 的原因如下：

- Lambdas 语法新奇，让 C 代码看起来像一种新语言，且多数人对函数式编程的理解不足，学习代价较大。
- Lambdas 本质上等价于定义一个仿函数，是一个语法糖，其并没有增加 C 的抽象能力。

# non-member begin() and end()

2021-06-16 20:48:57



全局函数 `std::begin()` 和 `std::end()` 可以更方便地抽象对容器的操作。例如：

```
int arr[] = {1,2,3};
std::for_each(std::begin(arr), std::end(arr), [](int n) {
  std::cout << n << std::endl;}
);
```

**危险**



**OceanBase禁止使用全局** std::begin() 和 std::end()

禁用原因：这个特性主要是让 STL 更加好用，然而 OceanBase 禁止使用 STL 容器。

# static_assert and type traits

2021-06-16 20:48:58



static_assert and type traits 特性指代编译器支持在编译期间 assert，以及编译期约束检查。例如：

```
template <typename T, size_t Size>
class Vector
{
  static_assert(Size < 3, "Size is too small");
  T _points[Size];
};
```

**说明**



OceanBase 允许使用 static_assert and type traits

允许原因：虽然 OceanBase 代码已经自己定义了`STATIC_ASSERT`，但是其只是对编译器检查的模拟，报错不友好。而 `type_traits` 给模板的使用带来很大好处。

# Move semantics

2021-06-16 20:48:58



move constructor 和 move assignment operator 是 C++11 最重要的一个新特性。伴随着它，引入了 rvalue 的概念。*移动*的语义可以让很多容器的实现变的比以前高效很多。例如：

```cpp
// move constructor
Buffer(Buffer&& temp):
    name(std::move(temp.name)),
    size(temp.size),
    buffer(std::move(temp.buffer))
{
  temp._buffer = nullptr;
  temp._size = 0;
}
```

**是否允许**

**危险**



OceanBase 禁止使用 move 相关语法。

禁用 Move semantics 主要基于以下考虑：

- OceanBase 不使用 STL 容器，所以标准库使用移动语义的优化对我们没有带来好处。
- Move semantic 和 rvalue 的语义比较复杂，容易引入坑
- 用它改造 OceanBase 现有某些容器，确实可以带了性能的提升。但是，OceanBase 的内存管理方式已经使得移动语义的用武之地变小了。很多时候，我们在实现的时候已经做了优化，在容器里只保存指针，而不是大对象。

建议在其他 C++11 特性熟悉一段时间以后，下一次修订编码规范的时候再考虑。

# constexpr

2021-06-16 20:48:59



`constexpr` 关键字提供了更加标准化的编译时常量表达式计算支持，不再需要使用各种模板的奇技淫巧来达到编译期计算的效果。例子：

```
constexpr int getDefaultArraySize (int multiplier)
{
  return 10 * multiplier;
}
int my_array[ getDefaultArraySize( 3 ) ];
```

**说明**



**OceanBase 允许使用 constexpr**

允许原因：常量对于编译优化总是更加友好的。在上面的例子中，还避免了宏的使用。此外，`constexpr` 支持浮点数计算，这是用 `static const` 无法代替的。

# Uniform initialization syntax and semantics

2021-06-16 20:49:00



该特性支持任意类型变量在任意上下文的初始化，都可以使用统一的{}语法。例如：

```
X x1 = X{1,2};
X x2 = {1,2};   // the = is optional
X x3{1,2};
X* p = new X{1,2};
```

**危险**



OceanBase 禁止使用 Uniform initialization syntax and semantics

禁用原因：该特性并没有带来显著的好处，同时又会显著影响 OceanBase 代码的风格，影响可读性。

# Right Angle Brackets

2021-06-16 20:49:00



原来 C 语言定义模板的模板嵌套的时候，结尾的 >> 之间必须用空格分离，该特性允许不使用空格。例如：

```
typedef std::vector<std::vector<bool>> Flags;
```

**说明**



OceanBase 允许使用该特性

# Variadic templates

2021-06-16 20:49:01



可变参数模板，能表示0到任意个数、任意类型的参数。例如：

```cpp
template<typename Arg1, typename... Args>
void func(const Arg1& arg1, const Args&... args)
{
process( arg1 );
func(args...); // note: arg1 does not appear here!
}
```

**说明**



OceanBase 允许使用 Variadic Templates

允许原因：因为没有变长模板参数，OceanBase 的一些基础库如 `to_string`，`to_yson`， RPC 框架，日志库等都需要用一些奇技淫巧结合宏来实现，而使用可变参数模板则更加安全。

# Unrestricted unions

2021-06-16 20:49:01



Unrestricted unions 允许 Union 中包含具有构造函数的类作为成员。例如：

```
struct Point {
  Point() {}
  Point(int x, int y): x(x), y(y) {}
  int x, y;
};
union U {
  int z;
  double w;
  Point p; // Illegal in C03; legal in C11.
  U() {} // Due to the Point member, a constructor definition is now needed.
  U(const Point& pt) : p(pt) {} // Construct Point object using initializer list.
  U& operator=(const Point& pt) { 
    new(&p) Point(pt); 
    return *this; 
  } // Assign Point object using placement 'new'.
};
```

**说明**



**OceanBase 允许使用 Unrestricted unions**

允许原因：OceanBase 的代码中，有多处因为 union 对于成员类型的限制，不得不定义了冗余的域，或者用很 trick 的方法绕过（定义 char 数组占位）。悲惨的例子见`sql::ObPostExprItem`。

# Explicitly defaulted and deleted special member functions

2021-06-16 20:49:02



该特性允许程序员显式地要求或者禁止编译器隐式生成构造函数，拷贝构造函数，赋值运算符，析构函数等。例如：

```
struct NonCopyable {
  NonCopyable() = default;
  NonCopyable(const NonCopyable&) = delete;
  NonCopyable& operator=(const NonCopyable&) = delete;
};
struct NoInt {
  void f(double i);
  void f(int) = delete;
};
```



**说明**



**OceanBase 允许使用该特性**

# Type alias (alias declaration)

2021-06-16 20:49:02



使用新的 alias declration 语法可以定义类型的别名，和之前的 `typedef` 类似；而且，还可以定义别名模板。例如：

```
// C++11
using func = void(*)(int);

// C++03 equivalent:
// typedef void (*func)(int);
template using ptr = T*;
// the name 'ptr' is now an alias for pointer to T
ptr ptr_int;
```

**危险**



OceanBase 禁止使用 alias declaration

禁止原因：暂时没有遇到别名模板的需求，而非模板的别名用 `typedef` 可以达到相同的作用。

# 总结

2021-06-16 20:49:03



1. **不允许使用智能指针**，允许通过 `Guard` 类自动释放资源。
2. 要求使用内存分配器申请内存，内存释放后要立即将指针置为 `NULL`。

1. **禁止使用** **`std::string`** **类，采用** **`ObString`** **代替**。另外，操作 C 字符串时，要求使用限长字符串函数。
2. **作为参数传递数组/字符串/缓冲区时必须同时传递长度，读写数组/字符串/缓冲区内容时要检查下标是否越界**。

1. 只能在同一个文件中使用友元，如果必须违反，请事先征得小组负责人的同意，并详细注释原因。将单元测试类声明为友元可以例外，但需谨慎使用。
2. **禁止使用 C++ 异常**。

1. 禁止使用运行时类型识别（RTTI）。
2. 使用 `static_cast<>` 等 C++ 类型转换，禁止使用类似`int y = (int) x`的 C 强制类型转换。

1. 尽量采用 `to_cstring` 输出。
2. 返回的 `ret` 错误码使用 `int`，函数参数和循环次数尽量使用 `int64_t`。其它情况使用指定长度的有符号数，例如 `int32_t`，`int64_t`。尽量避免使用无符号数。

1. 尽量使用 `sizeof(var_name)` 代替 `sizeof(type)`。
2. 整数用 `0`，实数用 `0.0`，指针用 `NULL`，字符串用 `’\0’`。

1. **除了已有的宏之外，不得定义新的宏，以内联函数、枚举和常量代替**。如果必须违反，请事先征得小组负责人的同意，并详细注释原因。
2. **除了 STL 中 <algorithm> 头文件定义的算法类函数外，禁止使用** **`STL`** **及** **`boost`**。**如果必须违反，请事先征得项目负责人和项目架构师的同意，并详细注释原因。**



# 命名规则

2021-06-16 20:49:04



## 通用规则

函数命名、变量命名、文件命名应该具有描述性，不要过度缩写，类型和变量应该是名词，函数可以用“命令性”动词。

标识符命名有时候会使用一些通用的缩写，但不允许使用过于专业或不大众化的。例如我们可以使用如下范围：

1. temp 可缩写为 tmp。
2. statistic 可缩写为 stat。
3. increment 可缩写为 inc。
4. message 可缩写为 msg。
5. count 可缩写为 cnt。
6. buffer 可缩写为 buf，而不是 buff。
7. current 可缩写为 cur，而不是 curr。

使用缩写时，需要考虑是否每个项目组成员都能理解。如果不太确定，避免使用缩写。

类型和变量一般为名词，例如，`ObFileReader`，`num_errors`。

函数名通常是命令性的，例如 `open_file()`，`set_num_errors()`。

## 文件命名

自描述良好的全小写单词组成，每个单词之间使用 ’_’ 分割，例如 `ob_update_server.h` 以及 `ob_update_server.cpp`。

`.h` 文件和 `.cpp` 文件互相对应，如果模板类代码较长，可以放到 `.ipp` 文件中，例如 `ob_vector.h` 和`ob_vector.ipp`。

## 类型命名

使用自描述良好的单词组成。为了和变量区分，建议使用单词首字母大写，中间无分隔符的方式。嵌套类不需要加 “Ob” 前缀，其它类都需要加 “Ob” 前缀。例如：

```cpp
// class and structs
class ObArenaAllocator
{ ...
struct ObUpsInfo
{ ...
 
// typedefs
typedef ObStringBufT<> ObStringBuf;
 
// enums
enum ObPacketCode
{ …
 
// inner class
class ObOuterClass
{
private:
 class InnerClass
 { …
};
```

接口类需要前面加 “I” 修饰符，其它类都不要加，例如：

```cpp
 class ObIAllocator
{ ...
}
```

## 变量命名

**类内变量命名**

自描述良好的全小写单词组成，单词之间使用 ’_’ 分隔，为了避免与其他变量混淆，要求采用后端加入 ’_’ 的方式区分，例如：

```cpp
class ObArenaAllocator
{
private:
 ModuleArena arena_;
};
 
struct ObUpsInfo
{
 common::ObServer addr_;
 int32_t inner_port_;
};
```

**普通变量命名**

自描述良好的全小写单词组成，单词之间使用 ’_’ 分隔。

**全局变量命名**

除了已有的全局变量外，不得使用新的全局变量。如果必须违反，请事先征得小组负责人的同意，并详细注释原因。全局变量由自描述良好的全小写单词组成，单词之间使用 ’_’ 分隔，为标注全局性质，要求在前端加入 ’g_’ 修饰符。例如：

```cpp
// globle thread number
int64_t g_thread_number;
```

## 函数命名

**类别内函数命名**

自描述良好的全小写单词组成，单词之间使用 ’_’ 分隔，例如：

```cpp
class ObArenaAllocator
{
public:
 int64_t used() const;
 int64_t total() const;
};
```

**存取函数命名**

存取函数的名称需要和类成员变量对应，如果类成员变量为 `xxx`，那么存取函数分别为 `set_xxx` 和 `get_xxx`。

**普通函数命名**

自描述良好的全小写单词组成，单词之间使用 ’_’ 分隔。

## 常量命名

所有编译时常量, 无论是局部的, 全局的还是类中的, 要求全部使用大写字母组成，单词之间以 ’_’ 分割。例如：

```cpp
static const int NUM_TEST_CASES = 6;
```

## 宏命名

**尽量避免使用宏，**宏命名全部使用大写字母组成，单词之间以 ’_’ 分割。注意，定义宏时必须对参数加括号。例如：

```cpp
//正确的写法
#define MAX(a, b) (((a) > (b)) ? (a) : (b))
//错误的写法
#define MAX(a, b) ((a > b) ? a : b)
```

## 注意事项

1. 尽量不要采用缩写，除非缩写名足够清晰且能被项目组成员广泛接受。
2. 除了接口类名称需要加 I 修饰，其它类、结构体、枚举类型都不需要修饰符。
3. struct 内的变量名也需要加下划线。

# 代码缩进

2021-06-16 20:49:04



不要使用 Tab 键进行缩进，可以用空格替代，不同的编码工具都可以进行设置，要求使用两个空格缩进（4 个空格缩进在单行代码比较长的时候会显得有些不够紧凑）。

# 空行

2021-05-31 23:09:40



尽量减少不必要的空行，只有代码逻辑明显分为多个部分时才这么做。

函数体内部可视代码决定，一般来讲，只有当代码逻辑上分成多个部分时，各个部分之间才需要增加一个空行。

下面这些代码都不应该有空行：

```
// 函数头、尾不要有空行
void function()
{
  int ret = OB_SUCCESS;
 
}
 
// 代码块头、尾不要有空行
while (cond) {
  // do_something();
 
}
if (cond) {
 
  // do_something()
}
```

下面的空行是合理的。

```
// 函数初始化和业务逻辑是两个部分，中间有一个空行
void function(const char *buf, const int64_t buf_len, int64_t &pos)
{
  int ret = OB_SUCCESS;
  int64_t ori_pos = pos;
  if (NULL == buf || buf_len <= 0 || pos >= buf_len) {
    // print error log
    ret = OB_INVALID_ARGUMENT;
  } else {
    ori_pos = pos;
}
 
  // 执行业务逻辑
  return ret;
}
```

# 行长度

2021-06-16 20:49:05



每行长度不得超过 100 个字符，一个汉字相当于两个字符。

100 个字符是单行的最大值，如下几种情况可以提高最大限制到 120 个字符：

1. 如果一行注释包含了超过 100 字符的命令或者 URL。
2. 包含长路径。

# 函数声明

2021-06-16 20:49:05



返回类型和函数名在同一行,参数也尽量放在同一行。

函数格式如下：

```cpp
int ClassName::function_name(Type par_name1, Type par_name2)
{
 int ret = OB_SUCCESS;
 do_something();
 …
 return ret;
}
```

如果同一行文本较多，容不下所有参数，可以将参数拆分到下一行，例如：

```cpp
int ClassName::really_long_function_name(Type par_name1,
   Type par_name2, Type par_name3) //空4格
{
 int ret = OB_SUCCESS;
 do_something();
 …
 return ret;
}
```

也可以将每个参数单独放一行，后面每个参数与第一个参数对齐，例如：

```cpp
//后面的参数和第一个参数对齐
int ClassName::really_long_function_name(Type par_name1,
                                                           Type par_name2, //和第一个参数对齐
                                                           Type par_name3)
{
 int ret = OB_SUCCESS;
 do_something();
 …
 return ret;
}
```

如果连第一个参数都放不下，采用以下格式：

```cpp
//后面的参数另起一行，空4格
int ClassName::really_really_long_function_name(
    Type par_name1, //空4格
    Type par_name2,
    Type par_name3)
{
 int ret = OB_SUCCESS;
 do_something();
 …
 return ret;
}
```

注意：

- 返回值总是和函数名在同一行。
- 左圆括号总是和函数名在同一行。
- 函数名和左圆括号间没有空格。
- 圆括号与参数间没有空格。
- 左大括号总是单独位于函数的第一行（另起一行）。
- 右大括号总是单独位于函数最后一行。
- 函数声明和实现处的所有形参名称必须保持一致。
- 所有形参应尽可能对齐。
- 缺省缩进为 2 个空格。
- 换行后的参数保持 4 个空格的缩进。
- 如果函数声明成 const, 关键字 const 应与最后一个参数位于同一行。



有些参数没有用到，在函数定义时将这些参数名，例如：

```cpp
//正确的写法
int ObCircle::rotate(double /*radians*/)
{
}
 
//错误的写法
int ObCircle::rotate(double)
{
}
```

# 函数调用

2021-06-16 20:49:06



函数调用代码尽量放在同一行，如果放不下，可以切成多行，切分方式与函数声明类似。

函数调用形式如下（左圆括号后和右圆括号前都不要留空格）：

```cpp
int ret = function(argument1, argument2, argument3);
```

如果切分成多行，可以将后面的参数拆分到下一行，格式如下：

```cpp
int ret = really_long_function(argument1,
    argument2, argument3); //空4格
```

也可以将每个参数单独放一行，后面每一行都和第一个参数对齐，格式如下：

```cpp
int ret = really_long_function(argument1,
                                         argument2, //和第一个参数对齐
                                         argument3);
```

如果函数名太长，可以将所有的参数独立成行，格式如下：

```cpp
int ret = really_really_long_function(
    argument1,
    argument2, //空4格
    argument3);
```

对于 placement new，new 与指针变量之间需要加一个空格，格式如下：

```cpp
new (ptr) ObArray();// new与‘(’之间包含一个空格
```

# 条件语句

2021-06-16 20:49:07



`{` 和 `if` 或者 `else` 在同一行，`}` 另起一个新行。另外，`if` 与 `(` 之间，`)` 与 `{` 之间，都保证包含一个空格。

条件语句往往是这样的：

```cpp
if (cond) { // (与cond，cond与)之间没有空格
 …
} else { // }和 else，else 和 { 之间分别有一个空格
 …
}
```

无论如何，`if` 和 `else` 语句都需要有 `{` 和 `}` ，即使该分支只有一行语句。原则上，`}` 总是另起一行，但是有一种情况可以例外。如果 `else` 分支什么也不做，`}` 不需要另起一行，如下：

```cpp
if (OB_SUCCESS == (ret = do_something1())) {
 …
} else if (OB_SUCCESS == (ret = do_somethng2())) {
 …
} else { } // else 分支什么也不做，} 不要求另起一行
```

对于比较语句，如果为 `=`，`!=`，那么，需要将常量写在前面；而`>`，`>=`，`<`，`<=`，则没有这个限制。例如：

```cpp
//正确的写法
if (NULL == p) {
 …
}
 
//错误的写法
if (p == NULL) {
…
}
```

# 表达式

2021-06-16 20:49:07



表达式运算符与前后变量之间各有一个空格，如下：

```cpp
a = b; // =的前后各有一个空格
a > b;
a & b;
```

对于布尔表达式，如果超过了行的最大长度，需要注意断行格式。另外，复杂表达式需要用括号明确表达式的操作顺序，避免使用默认优先级。

断行时，逻辑运算符总是位于下一行的行首，空4格：

```cpp
if ((condition1 && condition2)
    || (condition3 && condition4) // || 操作符位于行首，空4格
    || (condition5 && condition6)) {
    do_something();
    ……
} else {
    do_another_thing();
    ……
}
```

如果表达式比较复杂，应该加上括号明确表达式的操作顺序。

```cpp
//正确的做法
word = (high << 8) | low;
if ((a && b) || (c && d)) {
 …
} else {
 …
}
 
//错误的做法
word = high << 8 | low;
if (a && b || c && d) {
 …
} else {
 …
}
```

三目运算符尽量写成一行，如果超过一行，需要写成三行。如下：

```cpp
//三目运算符写成一行
int64_t length = (0 == digit_idx_) ? digit_pos_ : (digit_pos_ + 1)；
 
//三目运算符写成三行
int64_t length = (0 == digit_idx_)
    ? (ObNumber::MAX_CALC_LEN - digit_pos_ - 1) //空4格
    : (ObNumber::MAX_CALC_LEN - digit_pos_);
 
//错误：不允许分成两行
int64_t length = (0 == digit_idx_) ? (ObNumber::MAX_CALC_LEN – digit_pos_ - 1)
   : (ObNumber::MAX_CALC_LEN – digit_pos_);
```

# 循环和开关选择语句

2021-06-16 20:49:08



`switch` 语句和其中的 `case` 块都需要使用 {}。另外，每个 `case` 分支必须加入 `break` 语句。即使能够确保不会走到 `default` 分支，也需要写 `default` 分支。

```cpp
switch (var) {
case OB_TYPE_ONE: { //顶格
    …   //相对于case空4格，相对于switch空4格
    break;
}
case OB_TYPE_TWO: {
    …
    break;
}
default: {
    进行错误处理;
}
}
```

空循环体需要写一行 empty 注释，而不是一个简单的分号。例如：

```cpp
//正确的做法
while (cond) {
// empty
}
 
for (int64_t i = 0; i < num; ++i) {
 // empty
}
 
//错误的做法
while (cond) ;
for (int64_t i = 0; i < num; ++i) ;
```

# 变量声明

2021-06-16 20:49:08



每行只声明一个变量，变量声明时必须初始化。在声明指针变量或参数时，`*`、`&` 与变量名挨着。函数类型声明时，指针或引用 `*`、`&` 也是如此。

```cpp
//正确的做法
int64_t *ptr1 = NULL;
int64_t *ptr2 = NULL;
 
//错误的做法
 
int64_t *ptr1 = NULL, ptr2 = NULL; //错误，每行只声明一个变量
int64_t *ptr3; //错误，变量声明时必须初始化
int64_t* ptr = NULL; //错误，* 与变量名挨着，而不是与数据类型挨着
 
char* get_buf(); //错误，* 与变量名挨着，而不是与数据类型挨着
char *get_buf(); //正确
 
int set_buf(char* ptr); //错误，* 与变量名挨着，而不是与数据类型挨着
int set_buf(char *ptr); //正确
```

# 变量引用

2021-06-16 20:49:09



对于引用和指针，需要注意：句点（.）或箭头（->）前后不要有空格。指针 (*) 和地址操作符 (&) 之后不能有空格。

```
//正确的做法
p = &x;
x = *p;
x = r->y;
x = r.y;
```

# 预处理指令

2021-06-16 20:49:10



预处理指令不要缩进，从行首开始。即使预处理指令位于缩进代码块中，指令也应该从行首开始。

```cpp
//正确的写法，预处理指令位于行首
#if !defined(_OB_VERSION) || _OB_VERSION<=300
   do_something1();
#elif _OB_VERSION>300
   do_something2();
#endif
```

# 类格式

2021-06-16 20:49:10



声明次序依次是 `public`、`protected`、`private`，这三个关键字顶格，不缩进。

类声明的基本格式如下：

```cpp
class ObMyClass : public ObOtherClass // :的前后各有一个空格
{                                  // {另起一个新行
public:                               //顶格
  ObMyClass();                     //相对 public 缩进 2 格
  ~ObMyClass();
  explicit ObMyClass(int var);
 
  int some_function1();           //第一类功能函数
  int some_function2();
 
  inline void set_some_var(int64_t var) {some_var_ = var;} //第二类功能函数
  inline int64_t get_some_var() const {return some_var_;}
 
  inline int some_inline_func();   //第三类功能函数
 
private:
  int some_internal_function();     //函数定义在前
 
  int64_t some_var_;               //变量定义在后
  DISALLOW_COPY_AND_ASSIGN(ObMyClass);
};
 
int ObMyClass::some_inline_func()
{
 …
}
```

类的声明次序请参考第 4 章[声明次序](https://open.oceanbase.com/docs/oceanbase-code-style-guide/oceanbase-code-style-guide/V3.1.0/declaration-order)一节。

注意，只有实现代码为一行的 `inline` 函数可以放到类定义里面，其它 `inline` 函数放到 `.h` 文件中的类定义外面。上例中的 `set_some_var` 和 `get_some_var` 只有一行实现代码，因此放到类定义里面；`some_inline_func` 的实现代码超过一行，需要放到类定义外面。这样的好处是使得类定义更加紧凑。

# 初始化列表

2021-06-16 20:49:11



构造函数初始化列表放在同一行或者按照 4 格缩进并排成几行，且后面的参数和第一个参数对齐。另外，如果初始化列表需要换行的话，从第一个参数就要开始换行。

推荐您使用的初始化列表格式如下：

```cpp
//初始化列表放在同一行
ObMyClass::ObMyClass(int var):some_var_(var), other_var_(var+1)
{
 …
}
 
//初始化列表放在多行，按照 4 格缩进
ObMyClass::ObMyClass(int var):
    some_var_(var),
    some_other_var_(var+1) //第二个参数和第一个参数对齐
{
…
}
```

# 命名空间

2021-06-16 20:49:11



命名空间内容不要缩进。

```cpp
namespace oceanbase
{
namespace common
{
class ObMyClass // ObMyClass 不要缩进
{
 …
}
} // namespace common
} // namespace oceanbase
```

# 常量代替数字

2021-06-16 20:49:12



涉及物理状态或者含有物理意义的常量，不应直接使用数字，必须用有意义的枚举或常量，例如：

```cpp
const int64_t OB_MAX_HOST_NAME_LENGTH = 128;
const int64_t OB_MAX_HOST_NUM = 128;
```

# 注意事项

2021-06-16 20:49:13



1. `if` 和 `else`、`for` 和 `while`，以及 `switch-case` 语句的 `{` 都放在行的末尾，而不是另起一行。
2. 定义类的 `public`，`protected` 以及 `private` 关键字空 2 格，注意类的声明次序。
3. 将一行切割为多行时需要注意格式。
4. 尽量减少不必要的空行，只有代码逻辑明显分为多个部分时才这么做。
5. 命名空间的内容不要缩进。

# 注释

2021-06-01 08:00:48

注释是为了别人理解代码而写的，下面的规则描述了应该注释什么，注释在哪里。

## 注释语言与风格

注释语言可以使用英文，类注释也可以使用中文，注释风格采用//。

注释的目的是为了让其它人更容易理解你的代码，对于小块注释（例如函数注释或者函数内部实现注释），英文表达往往比较容易；对于大块注释，例如类注释，也许中文表达比较容易。因此，注释语言倾向于使用英文，然而，对于类注释，可以使用中文。

注释风格可以用//，也可以用/* */，除了头文件的注释，其它情况都采用//。

## 文件注释

在每一个文件开头加入版权公告, 版权公告见2.2节，然后是文件内容描述、著作人信息。统一采用以下格式：

```
// Copyright ...
// Author:
//   rizhao.ych@alipay.com
//   yubai.ych@alipay.com
//

// This file defines common macros...（此处加入文件功能、关键算法等说明）
```

对于关键性的算法及业务逻辑，应该在此处描述清晰，定义文件头部。

## 类注释

每个类的定义都要附带一份注释, 描述类的功能和用法。例如：

// memtable 迭代器：如下四个需求全部使用MemTableGetIter迭代

// 1. [常规get/scan] 需要构造RNE的cell，和根据create_time构造mtime/ctime的cell，如果有列过滤还会构造NOP的cell

// 2. [QueryEngine的dump2text] 没有列过滤和事务id过滤，不会构造NOP，但是会构造RNE/mtime/ctime

// 3. [转储] 没有列过滤和事务id过滤，会在QueryEngine跳过空的行，不会构造RNE和NOP，但会构造mtime/ctime

// 4. [单行merge] merge前需要判断事务id过滤后是否还有数据，如果没有就不调用GetIter迭代，防止构造出RNE写回memtable；此外还需要RowCompaction保证不

调整顺序，防止表示事务ID的mtime被调整到普通列的后面

// 5. [update and return] 与常规get/scan类似，但没有事务id过滤

class MemTableGetIter : public common::ObIterator

{

};

请注意在此处表明使用类需要注意的事项，尤其是是否线程安全、资源如何释放等等。

## 函数注释

**函数声明注释**

函数声明注释位于函数声明之前，主要描述函数声明本身而不是函数怎样完成，需要描述的内容包括：

l函数的输入输出.

l如果函数分配了空间,需要由调用者释放.

l参数是否可以为NULL.

l是否存在函数使用上的性能隐患.

l函数是否是可重入的，其同步前提是什么

// Returns an iterator for this table.

// Note:

// It’s the client’s responsibility to delete the iterator

// when it’s done with it.

//

// The method is equivalent to:

// ObMyIterator *iter = table->new_iterator();

// iter->seek_to_front();

// return iter;

ObMyIterator *get_iterator() const;

一般来讲，每个类的对外接口函数都需要注释。当然，构造函数、析构函数以及存取函数这样的自描述函数是不需要注释的。

如果注释需要说明输入、输出参数或者返回值，格式如下例：

// Gets the value according to the specified key.

//

// @param [in] key the specified key.

// @param [in] value the result value.

// @return the error code.

int get(const ObKey &key, ObValue &value);

函数可重入的注释示例如下：

// This function is not thread safe, but it will be called by only one xxx thread.

int thread_unsafe_func();

**函数实现注释**

如果函数实现算法比较独特或者有一些亮点，可以在.cpp文件中加入函数实现注释。例如使用的编程技巧, 实现的大致步骤, 或解释如此实现的理由, 比如说明为什么前半部分要加锁而后半部分不需要。注意此处的重点说明在于如何实现，而不是拷贝.h文件中的函数声明注释。

## 变量注释

局部变量可以不写注释。成员变量和全局变量一般都要写注释，除非项目组成员公认该变量是自描述的。如果变量的某些值有特殊含义，例如NULL，-1，那么，必须在注释中说明。

使用良好无歧义的语言标明变量用途、使用要点、作用范围。注释可以根据行字符多少决定出现在变量定义右侧或者变量定义顶部一行，例如：

//注释出现在顶部一行

private:

// Keeps track of the total number of entries in the table.

// -1 means that we don’t yet know how many entries the table has.

int num_total_entries_;



//注释出现在变量右侧

static const int NUM_TEST_CASES = 6; // the total number of test cases.

## 实现注释

同样，你必须在函数内部实现中对业务关键点、精巧算法、可读性差的部分进行详细注释。同样可以出现在代码段顶部或者某行代码右侧。

// it may fail, but the caller will retry until success.

ret = try_recycle_schema();

注意，不要以伪码方式写注释，那样过于繁琐且价值不大。

## TODO 注释

尚未实现或者未完美实现的功能，有时候我们需要加入TODO注释。所有的TODO注释必须体现工作人及完成时间，当然，如果完成时间未定，你可以明白的标注出来。例如：

// TODO(rizhao.ych): needs another network roundtrip, will be solved by 2014/12.

## 注意事项

1.注释语言可以使用英文，也可以使用中文，注释风格采用//

2.注释往往用于描述类、函数接口以及实现关键点。鼓励多写注释，除非是自描述的代码。

3.一定不要忘记TODO注释。



# 多线程

2021-06-16 20:49:13



## 起线程和停线程

1. 除了极特殊的情况,禁止动态起线程和停线程, `server` 一旦初始化完成，线程数就是固定的。特殊情况: 为 `server` 留的后门，在 `server` 所有线程都被占住的情况下增加一个工作线程。
2. 为了保证退出时某个线程不会忙等在一个死循环上，所以循环一般都要判断 `stop` 标志。

## pthread_key

1. `pthread_key` 最多只有 1024 个，并且这个限制不能调大，使用时需要特别注意。
2. 如果要使用大量的线程局部变量，推荐使用线程编号做数组下标获取一个线程私有的变量。OceanBase 中封装了一个 `itid()` 函数来获取连续递增的线程编号。

```cpp
void*get_thread_local_variable()
{
  return global_array_[itid()];
}
```

## 定时器

不能在定时器中完成耗时过长的任务,耗时长的任务需要提交给线程池执行。

## 加锁和解锁



1. 推荐使用 `Guard` 的方式使用锁

```cpp
//作用域是整个函数
int foo()
{
  SpinLockGuardguard(lock_);
  ...
}
//作用域是一个子句
while(...) {
  SpinLockGuardguard(lock_);
  ...
}
```

\2. 如果锁的范围不是整个函数或某个子句，比如在函数执行中途加锁，函数退出之前解锁,这种情况允许手工加锁和解锁:

```cpp
int foo()
{
  int ret = OB_SUCCESS;
  bool lock_succ = false;
  if (OB_SUCCESS!= (ret = lock_.lock())) {
    lock_succ = false;
  } else {
    lock_succ = true;
  }
  … //执行了若干语句
  if (lock_succ) {
    lock_.unlock();
  }
  return ret;
}
```

## cond/signal 的标准用法

1. 通过 `tbsys` 封装的 `CThreadcond` 使用 `cond`/`signal`
2. 禁止使用不带超时的 `cond_wait()`
3. 按以下的惯用法使用 `cond`/`signal`

```cpp
//等待的逻辑
cond.lock();
while(need_wait()){
  cond.wait(timeout);
}
cond.unlock();
//唤醒的逻辑
cond.lock();
cond.signal();
cond.unlock();
```

## 原子操作

统一使用定义在 `ob_define.h` 中的宏做原子操作,需要注意:

1. 原子读写也需要用 `ATOMIC_LOAD()` 和 `ATOMIC_STORE()` 完成
2. 用 `ATOMIC_FAA()`和 `ATOMIC_AAF()` 区分 `fetch_and_add` 和 `add_and_fetch`
3. 用 `ATOMIC_VCAS()` 和 `ATOMIC_BCAS()` 区分 CAS 操作返回 value 或 bool

## 编译器 barrier

一般要使用编译器 barrier 的地方也需要 memory barrier，并且 memory barrier 蕴含了编译器 barrier，所以应该没有什么地方需要使用编译器 barrier。

## memory barrier

1. 虽然有各种 memory barrier，但是我们只推荐使用 full barrier。因为更精细的 barrier 非常容易出错，目前在 OceanBase 的工程实践中也没有遇到必须用更精细的 barrier 才能满足性能的要求的代码。各种barrier有多复杂，可以参考这个文档:

   https://www.kernel.org/doc/Documentation/memory-barriers.txt

2. 原子操作自带 barrier，所有一般不需要手工加 barrier.

3. 如果需要手工加 barrier，使用宏:

\#defineMEM_BARRIER() __sync_synchronize()

## 引用计数和 shared_ptr

首先，不得使用 `shared_ptr`，因为 `shared_ptr` 只是语法糖，并没有解决我们希望使用引用计数解决的问题。简单而言，多线程同时操作不同的 `shared_ptr` 是安全的，但是多线程同时操作同一个 `shared_ptr` 是不安全的。当我们考虑引用计数的时候，往往都是需要多线程操作同一个 `shared_ptr`。具体可以参考 [http://en.cppreference.com/w/cpp/memory/shared_ptr ](http://en.cppreference.com/w/cpp/memory/shared_ptr)。

其次，引用计数难以实现正确。例如，在对引用计数加1之前，对象就有可能被回收或重用。



目前 OceanBase 中有 2 种方法使用引用计数，可以参考:

1. 下面这种简单的场景是可以使用引用计数的:

1. 1. 单线程构造对象，对象的初始引用计数为 1
   2. 之后单线程加引用计数，并把对象传给其余的线程使用，其余的线程使用完之后减引用计数。
   3. 最后单线程决定释放对象，把引用计数减 1。

OceanBase 中的 `FifoAllocator` 就是这种用法。

\2. 如果不满足上面的简单场景，需要用全局锁保证安全性:

1. 1. 在例子1中的第1步加读锁
   2. 在例子1中的第3步加写锁

UPS 管理 `schema_mgr` 时就是这么做的。

## 对齐

为了避免cache false sharing，如果一个变量会被多线程频繁访问，定义变量时推荐按 `cache line` 对齐。

```cpp
int64_t foo CACHE_ALIGNED;
```



但是如果某个对象大量存在，为了节省内存，允许不按 `cache line` 对齐。

如果是通过动态申请内存构造的对象，需要注意至少让对象的起始地址是 8 字节对齐的。比如使用 `page_arena` 时， 可以通过 `alloc_aligned()` 来分配 8 字节对齐的内存。

```cpp
struct_A *p= page_arena_.alloc_aligned(sizeof(*p));
```

## volatile

总的来讲，不推荐使用 volatile 变量，原因参考 [volatile 文档](https://www.kernel.org/doc/html/latest/process/volatile-considered-harmful.html)。

改用 `ATOMIC_LOAD()` 或 `ATOMIC_STORE()` 保证对变量的读写不会被优化掉。

```cpp
//错误的方法
volatileint64_ti= 0;
x = i;
i = y;
//推荐的做法
int64_ti= 0;
x = ATOMIC_LOAD(&i);

ATOMIC_STORE(&i, y);
```

在少数情况下使用 `volatile` 依然是合理的，比如用来指示状态，但是这个状态变化又没有严格时序上的意义，如指示线程退出的标志变量：

```
volatile bool stop_CACHE_ALIGNED;
```

或者某些监控项：

```
volatile int64_t counter_CACHE_ALIGNED;
```

## 使用 CAS 的方法

因为 `ATOMIC_VCAS` 在操作失败的情况下返回了 `*addr` 的最新值，所以每次重试的时候不必要再次用`ATOMIC_LOAD` 读取。

比如要实现原子加 1，按如下的方式使用 `CAS` 操作：

```cpp
int64_t tmp_val = 0;
int64_t old_val = ATOMIC_LOAD(addr)
while(old_val != (tmp_val = ATOMIC_VCAS(addr, old_val, old_val + 1))){
  old_val = tmp_val;
}
```

## spin wait 和 PAUSE

在 `spin wait` 的循环中要加入 `PAUSE()`，在某些 CPU 上 `PAUSE()` 可以提高性能，并且一般来讲 `PAUSE()` 可以降低 CPU 功耗。

```cpp
while(need_retry()) {
  PAUSE();
}
```

PAUSE 的作用可以看这个回答:http://stackoverflow.com/questions/12894078/pause-instruction-in-x86/12904645#12904645

## 临界区

不得在临界区执行耗时较长或者复杂的操作，例如打开/关闭文件，读写文件等。

## 避免程序 core 或退出

数据库系统重启的时间常常以小时计，大面积的 core 或者退出将导致数据库服务中断，并可能被恶意攻击者利用。因此必须避免程序 core 或者退出，例如访问空指针指向的地址（临时修改用于定位 bug 除外），或者调用 abort(除非收到外部指令)等。**如果必须违反，****请事先征得项目负责人和项目架构师的同意，并详细注释原因**。

# 日志规范

2021-06-16 20:49:14



1.0版本的日志模块有两项主要改进：

**1. 支持多维度、细粒度的打印级别设置**

相比之前版本只支持全局统一设定日志级别的情况，1.0 版本支持语句、session、租户和全局（或 server）四个不同范围的打印日志设置。其中不同范围的设置方式分别为：

- SQL 语句 hint
- 设置 session 日志级别变量
- 设置租户日志级别变量
- 设置系统日志级别变量

另外，1.0 版本还支持日志模块、子模块的概念。程序中打印日志时，需要指明该条日志所属的模块（或者模块 + 子模块）和该条日志所属的日志打印级别。系统支持用户为各模块、子模块的分别设置不同的打印级别。

**2. 更严格的日志打印格式**

0.5 版本中存在着打印日志格式不统一，不易读的问题。例如打印值为 5 的变量 m 时，有多种不同的打印格式：”m = 5”，“m=5”，“m(5)”，“m is 5”，“m:5” 等。新的日志模块提供用户按照 key-value 对的方式打印所需变量的值。

## 日志打印级别

| **级别**                     | **级别定义**                                                 |
| ---------------------------- | ------------------------------------------------------------ |
| **ERROR**                    | 任何未预期的、不可恢复的、需要人工干预的错误                 |
| **WARNING**                  | 可预期的并且可以由程序解决的异常情况                         |
| **INFO****（启动默认级别）** | 少量对系统状态变化的标记信息。例如，添加了某用户、某表、系统进入每日合并、partition 迁移等。 |
| **TRACE**                    | 请求粒度的调试信息，例如执行一条 SQL 语句的不同阶段分别打印一条 TRACE 日志 |
| **DEBUG**                    | 一般性的、详细的调试信息，用以跟踪系统内部的状态、数据结构等。 |

需要注意的是，DEBUG 日志往往用于集成测试或者线上系统的调试，不能用来代替单元测试。

## 打印模块的划分（实例）

| **模块**        | **子模块定义**                                      |
| --------------- | --------------------------------------------------- |
| **SQL**         | Parser, transformer, optimizer, executor, scheduler |
| **STORAGE**     | TBD                                                 |
| **TRANSACTION** | TBD                                                 |
| **ROOTSERVER**  | TBD                                                 |
| **COMMON**      | TBD                                                 |
| **DML**         | TBD                                                 |

各模块下的子模块定义将由各组内部进一步细化。模块与子模块的定义放在文件 `ob_log_module.h` 中。

## 打印范围的设置

1.0 版本支持用户按照语句、session 和全局（系统）范围分别设置打印级别。

系统中参考的优先级为语句、session、系统全局（或 server），只有在前一项无设置或设置无效的情况下，系统才会参考后面的级别设置。

**语句范围打印级别设置**

【设置格式】

语句 hint 中加入 **/\*+ … log_level=[log_level_statement]…\*/** （`log_level_statement` 的格式参见后面章节）

【作用范围】

整个语句的处理、执行过程，包括语句分析、优化、执行等等。语句执行结束后，该设置自动失效。



**session 范围打印级别设置**

【设置格式】

**`sql> set @@session.log_level = ‘[log_level_statement]’;`**

【作用范围】

自设置起到该 session 结束前。



**租户范围打印级别设置**

【设置格式】

**`sql>set @@global.log_level =‘[log_level_statement]’;`**

【作用范围】

从用户设置开始对所有用户 session 生效，直至用户所有 session 退出。



**系统（或 server）范围打印级别设置**

【设置格式】

**`sql>alter system set log_level =‘[log_level_statement]{,server_ip=xxx.xxx.xxx.xxx}’;`**

【作用范围】

当用户指定 server_ip 时，该次设定只对该 server 生效，并直到该 server 退出或重启前一直有效；当用户没有指定 server_ip 时，设定对整个系统内所有server生效，并保持到整个系统reboot之前（新上线的server也需服从该设定）。

**log_level_statement 格式**

***log_level_statement =\***

**mod_level_statement {, mod_level_statement }**

***mod_level_statement =\***

**[mod[.[submod|\*]]:][ERROR|WARNING|INFO|TRACE|DEBUG]**

其中 mod 和 submod 的定义参见[打印模块的划分（实例）](https://open.oceanbase.com/docs/oceanbase-code-style-guide/oceanbase-code-style-guide/V3.1.0/log-specifications#section-lcb-w89-b6s)。没有指明 mod 或 submod 的情况下，该设定对所有 mod 生效。如果多项 mod_level_statement 设定有冲突时，以最后一个有效的设定为准。

用户设定不保证原子性：例如，在存在多项设定时，如果第 n 项设定不成功（语法错误或是模块不存在），如果是 session 或是系统级设置则语句报错，但之前生效项不回滚，如果是在语句hint中发生，则不报错且之前生效项不回滚。

## 日志格式的统一

1.0版本统一使用 “*key=value*“ 的格式打印日志。由日志模块统一提供类似如下接口：

**OB_MOD_LOG(mod,submod, level, “info_string”, var1_name, var1, var2, 2.3, current_range, range,…);**

相应的打印信息为

**[2014-10-0910:23:54.639198] DEBUG ob_tbnet_callback.cpp:203 [12530][Ytrace_id] info_string(var1_name=5, var2=2.3, current_range= "table_id:50,(MIN;MAX)" )**

其中 info_string 为该条日志的主要信息总结，应简洁、明了、易读。避免出现 “operator failed“ 等无信息量的字符串。

每一行的日志头（包括文件名、行号等信息）由日志打印模块自动生成。为方便使用，日志模块头文件 (`ob_log_module.h`) 还将提供以模块、子模块为单位定义的宏，使在某一文件或是文件夹下的程序打印语句更为简洁，例如：

*`#define OB_SQL_PARSER_LOG(level`**`，`**`…) OB_MOD_LOG(sql, parser, level…)`*

打印变量的名称的选择应考虑不同场合的需求。如果不使用变量名本身，应考虑系统中是否已有相同含义的变量名在使用（如版本号打印为 ”data_version” 还是 ”version”，还是 ”data version”应该尽量统一），方便日后的调试和监控。

遇有因操作不成功而返回的地方必须打印日志，且必须打印该错误码。

新的日志由于支持模块、范围的设置，对打印信息的过滤将更加有效，原则上必要的 debug 日志信息应进一步丰富，以方便日后的排错和调试。

