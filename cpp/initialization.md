# 初始化 (Initialization)

- 初始化的形式
  - Direct-initialization
  - Copy-initialization
- 初始化的含义
- Value initialize
- Default initialize
- Zero initialize
- List initialize
  - narrowing conversion
- 字符数组初始化
- 聚合（aggregate）初始化
  - Aggregate
- 引用初始化

基于[c++20 draft n4868](https://timsong-cpp.github.io/cppwp/n4868/)

## 初始化的形式

通常讲到初始化，会想起变量的时候给的初始化值(initializer)。但是除此之后，还有一些地方会发生对象的初始化。所有的这些初始化操作遵从同一套规则。

初始化按照其形式，可以分为两大类，direct-initialization与copy-initialization

### Direct-initialization

- 变量定义中初始化值无`=`
  - `int i{1}; std::vector<std::string> vec(3);`
- new-initializer
  - `int *p = new int(3);`，使用`3`初始化一个新建的`int`对象
- `static_cast`
  - `static_cast` 会使用其参数初始化一个结果类型的对象，如：`static_cast<int>(2.0);`，其中使用`2.0`初始化一个`int`对象。
- 函数调用像是的类型转换
  - `int(2.0)`，其中使用`2.0`初始化一个`int`类型的对象
- condition 中的 brace-init-list
  - `if (Derived *pDerived{dynamic_cast<Derived*>(pBase)})`，使用 `dyanmic_cast<Drived*>(pBase)` 初始化 `pDerived`

### Copy-initialization

- 变量定义中使用 `=` 形式初始化
  - `int i = 1;` ，使用 `1` 初始化 `i`
- condition 中使用 `=` 形式初始化
  - `if (Derived *pDerived = dynamic_cast<Derived*>(pBase))`，使用 `dyanmic_cast<Drived*>(pBase)` 初始化 `pDerived`
- 参数传递，函数返回值

  - 示例：

    ```cpp
    int foo(int i) {
      return i+1;
    }
    void bar() {
      foo(1);
    }
    ```

    使用 `1` 初始化 `foo` 的参数 `i`，使用 `i+1` 初始化函数的返回值（右值）
- 抛出异常，异常捕获

  - 示例：

    ```cpp
    void foo() {
      std::string s{123};
      try {
        throw s;
      } cache (string &es) {
      }
    }
    ```

    在 `throw s;` 中，使用 `s` 初始化一个临时对象。在 `cache (string &es)` 中，临时对象的左值用于初始化 `es`。这两个初始化都是 copy initialize
- 聚合成员初始化。在聚合初始化中，每个成员的初始化都是 copy initialization
  - `std::string str[]{"abc", "def"}`，使用 `"abc"` 初始化 `str[0]` ，使用 `"def"` 初始化 `str[1]` 都是 copy-initialize 。注意对 `str` 的初始化使用的是 direct-initialize 的形式。

## 初始化的含义

根据被初始化对象的类型与初始化的形式不同，初始化会有不同的含义。具体初始化含义见下。copy-initialize 与 direct-initialize 均使用如下定义。

- 如果 initializer 是一个 *braced-init-list* 或者 `=` *branced-init-list* ，执行列表初始化
- 如果目标类型是引用类型，则执行引用初始化
- 如果目标类型是 `char`, `singed char`, `unsigned char`, `char8_t`, `char16_t`, `char32_t`, `wchar_t` 的数组，initializer 是一个 *string-literal* ，执行字符数组初始化
- 如果 initializer 是 `()` ，执行 value-initialze
- 如果目标类型是数组，initializer 应该是 `(` *pxpression-list* `)`，且其中的元素个数不应比数组长度更多。数组中的元素将依次由 *epression-list* 中的值 copy-initialize。对没有提供值的元素（数组长度比 *expression-list* 长），进行 value-initialize
- 如果目标类型是类类型(class type):
  - 如果 initializer 是一个右值并且与目标类型相同（忽略 cv-qualifier），则 initializer 用于初始化目标对象
  - 如果 1) 这是 direct initialization ；或者 2) 这是 copy initialization 且源类型与目标类型相同，或者是目标类型的派生类，那么使用构造函数初始化
    - 如果可以找到唯一可用的构造函数，那么该构造函数用于初始化对象
    - 如果没有可用的构造函数，且目标类型是一个聚合(aggregate class)，intializer 为 `(` *expression-list* `)` ，则使用 *expression-list* 中的值依次 copy initialize对应元素。未初始化值的元素执行 value initialize。
    - 否则，程序非法
  - 否则，尝试用户定义类型转换（包括构造函数）。不能找到合适的类型转换的程序非法。类型转换的结果将被用于 direct initialize 目标对象。
- 否则，如果源类型是一个类类型，将尝试用户定义类型转换。不能找到合适的类型转换的程序非法。
- 否则，如果这个是一个 direct initialization ，原类型是 `std::nullptr_t`，目标类型是 `bool` ，则目标对象被初始化为 `false`
- 否则， 如果需要，可以使用一个标准转换序列(stand conversion sequence)将源对象转换为目标类型。无法转换则程序非法。

## Value initialize

对一个类型 value initialize 含义是：

- 如果其为类类型，则
  - 如果该类没有默认构造函数，或者有用户提供的（user-provided）或 deleted 默认构造函数，该对象被 default initialize
  - 否则，对象被 zero initialize；并且，如果该类有 non-trivial 默认构造函数，该对象被 value initialize。
    - 例如

      ```cpp
      #include <string>
      struct A{
        int a;
        std::string s;
      };
      int main()
      {
        A a_obj{}; // a_obj.a == 0
      }
      ```

      类 `A` 有默认构造函数（自动生成的），该默认构造函数不是 user-provided ，也不是 deleted，因而进入该条处理。
      `a_obj` 被 zero initialize 。另外，由于其默认构造函数是 non-trivial （它含有个一个 `std::string` 成员），因而被进一步默认构造。
      由于有 zero initialize 存在，`a_obj.a == 0`。
- 如果目标对象类型为数组，则数组的每一个元素被 value initialize
- 否则，对象被 zero initialize

## Default initialize

对一个类型 default initialize 含义是：

- 对类类型，调用默认构造函数。
- 对数组类型，每一个元素被 default initialize
- 其他，不执行初始化操作

## Zero initialize

对一个类型 default initialize 含义是：

- scalar type 初始化为 0
- 对非 union 类类型，其 padding 被初始化为 0，非静态数据成员，基类 被 zero initialize
- 对 union ，padding 被初始化为 0 ，第一个非静态成员被 zero initialize
- 对数组，每一个元素被 zero initialize
- 对引用，不进行初始化

## List initialize

- 如果 *brace-init-list* 包含 *designated-initializer-list* ，目标类型必须为聚合（aggregate class），其中成员的出现顺序需要与定义顺序一致。执行聚合初始化（aggregate initialize）
- 如果目标是聚合类（aggregate class），initializer list 仅包含一个元素，其类型与目标类型相同或者是其派生类，那么使用该元素初始化目标对象。
- 如果目标类型是数组，initializer list 仅包含一个元素，且为合适类型的字符串字面量，则执行字符数组初始化。
- 如果目标类型是聚合类型（aggregate class），执行聚合初始化
- 如果initializer list 为空，目标类型为有默认构造函数的类类型，目标对象被 value initialize
- 如果目标类型是 `std::initializer_list<E>` ：创建一个 `const E[N]` 类型的数组，其中每一个元素用 initializer list 中的元素 copy initialize ，目标对象将引用这个数组。（如果任何一个元素初始化是需要 narrowing conversion，则程序非法）
- 如果目标类型是类类型，使用构造函数初始化。（如果参数初始化时需要 narrowing conversion，程序非法）
- 如果目标类型是有确定 underlying type 的枚举，intiiailizer list 仅含有一个元素，该元素可以被隐式转换为目标类型的 underlying type ，并且该初始化是 direct initialization，则目标对象的值为源对象转换为目标类型的 underlying type 后的值。（不可以是 copy initialize；如果需要 narrawing conversion ，程序非法）
- 如果 initialize list 有唯一元素，目标类型不为引用或者为一个 reference related to 源类型的引用，目标对象（或引用）由该元素初始化（如果需要 narrowing conversion，程序非法）
- 如果目标类型是引用，则用 initializer 执行 copy list initialize 初始化一个被引用类型的 prvalue，目标引用由该 prvalue direct initialize
- 如果 initializer list 为空，目标对象被 value initialize
- 其他，程序非法

### narrowing conversion

narrowing conversion 包括：

- 从浮点类型到整形的转换
- 从 `long double` 到 `double` 或 `float`，从 `double` 到 `float` 的转换，除非源值为常量表达式，且其在目标类型中可表示（包括可表示但无法精确表示的情况）
- 从整形或 unscoped enueration type 到浮点类型的转换，除非源值是一个常量表达式，且转换结果在目标类型可表示，且将转换结果再转换为源类型时其值不变
- 从整形或 unscoped enueration type 到整型的转换，该整型无法表示源类型中的所有值，除非源值是一个常量表达式，其值在经过 integral promotion 之后在目标类型可表示
- 从指针或指向成员的指针类型到 `bool`

## 字符数组初始化

字符数组可以用字符串字面量初始化。字符数组的类型需要与字符串字面量类型匹配。字符数组的中的元素由字符串字面量中的对应元素进行初始化。

字符串字面量（包括最后的 `'\0'`）不能比字符数组更长。如果字符串字面量较短，多余的数组元素将被 zero initialize 。

## 聚合（aggregate）初始化

### Aggregate

满足以下条件的数组或类（union 也是一种类）称作 aggregate:

- 没有声明构造函数，或继承（注：使用 `using`）构造函数
- 没有私有或保护的非静态数据成员
- 没有虚函数
- 没有虚的、私有的或被保护的基类

aggregate 的成员指：

- 对于数组，指它的每一个元素
- 对于类，指按声明顺序的每一个直接基类，以及按声明顺序的每一个非静态数据成员（如果类中有匿名联合（union），匿名联合本身是 aggregate 的一个成员，但匿名联合的成员不是）

对于显式提供了值的成员，将使用该值进行 copy initialize。如果需要 narrowing conversion，程序非法。对于匿名联合（anonymous union），只能对其中一个成员提供初始化。

对于非 union 的 aggregate 中没有被显式初始化的成员：

- 如果成员声明了 default member initializer，使用该值初始化
- 如果该成员不是引用，使用 `{}` copy initialize
- 否则，程序非法

当一个 union aggregate 被使用 `{}` 初始化时：

- 如果任意成员（variant member）有 default member initializer，使用该值初始化这一成员
- 否则，union 的第一个成员使用 `{}` 初始化

aggregate 的成员如果也是 aggregate ，子 aggregate 的初始化列表（在某些情况下）可以省略 `{`, `}`。空 `{}` 不能省略。如果省略，那么接下来的 initializer 全部被用于初始化子 aggregate ，直到子 aggregate 所有成员都有了 initializer 。剩余的 initializer 用于初始化父 aggregate 的剩余成员。

## 引用初始化

先要介绍两个概念：

- 类型 *cv1* `T1` 与 *cv2* `T2` 是 reference related ，表示 `T1` 与 `T2` 相似（约等于相同），或 `T1` 是 `T2` 的基类。
- 类型 *cv1* `T1` 与 *cv2* `T2` 是 reference compitable 表示指向 *cv2* `T2` 的指针可以通过一个标准转换序列（standard conversion sequence）转换为指向 *cv1* `T1` 的指针。（约等于 *cv1* > *cv2*，并且 `T1` 与 `T2` 相同，或者是 `T2` 的（可访问的且无歧义的）基类）

使用 *cv2* `T2` 的值初始化 *cv1* `T1` 的引用规则如下：

- 目标是左值引用并且满足以下条件之一：
  - initialize 为左值，*cv1* `T1` 与 *cv2* `T2` 是 reference compitable，引用绑定到该值（或其基类子对象）
  - `T2` 是类，但 `T1` 与 `T2` 并非 reference related，但 `T2` 可以转换为一个 *cv3* `T3` 的左值，且 *cv1* `T1` 与 *cv3* `T3` 是 reference compitable 的，引用绑定到转换后的 `T3` 的左值（或其基类子对象）
- 否则，如果目标为非常量左值引用，或者 volatile 左值引用，程序非法
- 否则：（以下为绑定至右值，prvalue 会被转换为 glvalue）
  - 如果初始化值为一个右值（非位域），或一个函数，且 *cv1* `T1` 与 *cv2* `T2` 是 reference compitable，引用绑定到该值（或其基类子对象）
  - `T2` 是类，但 `T1` 与 `T2` 并非 reference related，但 `T2` 可以转换为一个 *cv3* `T3` 的右值（或函数），且 *cv1* `T1` 与 *cv3* `T3` 是 reference compitable 的，引用绑定到转换后的 `T3` 的 glvalue（或其基类子对象）
- 否则：
  - 如果 `T1` 或 `T2` 是类，且 `T1` 与 `T2` 并非 reference related，则尝试通过自定义转换使用初始化值初始化一个 *cv1* `T1` 的对象（如不能初始化相应的对象，则程序非法）。转换的结果将用于初始化目标引用。
  - 其他情况，初始化值将被隐式转换为 *cv1* `T1`。引用绑定值转换的结果。
    - 如果 `T1` 与 `T2` 是 reference related，那么 *cv1* 应大于 *cv2* ，不能用 lvalue 初始化 rvalue reference 。
  
  例如：

  ```cpp
  struct Banana { };
  struct Enigma { operator const Banana(); };
  struct Alaska { operator Banana&(); };
  void enigmatic() {
    typedef const Banana ConstBanana;
    Banana &&banana1 = ConstBanana(); // error
    Banana &&banana2 = Enigma(); // error
    Banana &&banana3 = Alaska(); // error
  }
  const double& rcd2 = 2; // rcd2 refers to temporary with value 2.0
  double&& rrd = 2; // rrd refers to temporary with value 2.0
  const volatile int cvi = 1;
  const int& r2 = cvi; // error: cv-qualifier dropped
  struct A { operator volatile int&(); } a;
  const int& r3 = a; // error: cv-qualifier dropped
  // from result of conversion function
  double d2 = 1.0;
  double&& rrd2 = d2; // error: initializer is lvalue of related type
  struct X { operator int&(); };
  int&& rri2 = X(); // error: result of conversion function is lvalue of related type
  int i3 = 2;
  double&& rrd3 = i3; // rrd3 refers to temporary with value 2.0
  ```
  