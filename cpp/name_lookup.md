# 名字查找

[C++](https://timsong-cpp.github.io/cppwp/n4868) 有一个原则，就是标识符要先声明，再使用。每一个标识符在使用的时候，都要找到它对应的声明。这个过程，就是名字查找。

在程序的不同结构中，名字查找的规则也不尽相同。下面简述如下。

## Unqualified name lookup

简单地说，就是单独一个标识符的查找。

### 名字空间中对标识符的使用

名字空间中的使用的标识符需要在使用前，在当前或包含它的名字空间中被声明。

### 名字空间中定义的函数中

这包括函数名之后出现的任何符号，包括参数表中出现的类型、默认参数等。

其声明需要出现在使用之前，依次查找当前块（block），包含当前块的块，函数属于的 namespace，该 namespace 的父 namespace。

```cpp
namespace A {
    namespace N {
        void f();
    }
}
void A::N::f() {
    i = 5;
    // i 的声明会在以下地方被查找:
    // 1) A::N::f 中，i 的只用之前
    // 2) namespace N
    // 3) namespace A
    // 4) 全局名字空间，A::N::f 的定义之前
}
```

### 类定义中的名字

类定义中的名字按其是否在一个 complete-class context ，查找跪在不同。

Complete-class context 是指：函数体、函数的默认参数、noexcept-specifier，成员初始化

在 complete-class context 外，声明要在使用之前，查找范围包括：当前类、基类的成员、包含当前类定义的类（如有）或其基类成员、包含当前类的函数（如有）、包含当前类的namespace。

```cpp
namespace M {
    class B { };
}
namespace N {
    class Y : public M::B {
        class X {
            int a[i];
        };
    };
}
// 会在以下位置查找 i 的声明:
// 1) class N::Y::X 中，i 使用之前
// 2) class N::Y 中, N::Y::X 定义之前
// 3) N::Y’s 的基类 M::B
// 4) 名字空间 N, N::Y 定义之前
// 5) 全局名字空间, N 定义之前
```

在 complete-class context 中，当查找类成员时，并不要求成员声明在标识符使用之前，查找范围包括：当前块或包含的块（如有）、当前类成员或基类的成员、包含当前类定义的类（如有）的成员或其基类成员、包含当前类的函数（如有）、包含当前类的namespace。

```cpp
class B { };
namespace M {
    namespace N {
        class X : public B {
            void f();
        };
    }
}
void M::N::X::f() {
    i = 16;
}
// 会在以下位置查找 i 的声明:
// 1) M::N::X::f, i 的使用之前
// 2) class M::N::X
// 3) M::N::X 的基类 B
// 4) namespace M::N
// 5) namespace M
// 6) global scope, M::N::X::f 定义之前
```

### friend

友元函数如果定义在类中，那么其定义的名字查找规则与类成员一致。

如果友元函数时类成员，那么标识符将首先在该函数所在的类中查找，除非是用于 *declarator-id* 中的 *template-argument* 的标识符。

```cpp
struct A {
    typedef int AT;
    void f1(AT);
    void f2(float);
    template <class T> void f3();
};
struct B {
    typedef char AT;
    typedef float BT;
    friend void A::f1(AT); // parameter type is A::AT
    friend void A::f2(BT); // parameter type is B::BT
    friend void A::f3<AT>(); // template argument is B::AT
};
```

### 默认参数、*mem-initializer*

在默认参数、*mem-initializer* 的 *expression* 中，函数参数是可见的，并且会覆盖外层同名声明。

### 枚举

枚举定义中，已定义的 enumerator 在后续定义中可见

### static member

类的静态成员定义中，名字查找规则同类成员函数

### 名字空间的数据成员

如果名字空间的数据成员的定义出现在名字空间之外，其定义中的标识符查找方式按照其定义在其名字空间中时的规则进行。

```cpp
namespace N {
    int i = 4;
    extern int j;
}
int i = 2;
int N::j = i; // N::j == 4
```

## Argument-dependet name lookup

此规则用于函数调用。

```cpp
typedef int f;
namespace N {
    struct A {
        friend void f(A &);
        operator int();
        void g(A a) {
            int i = f(a); // f is the typedef, not the friend function: equivalent to int(a)
        }
    };
}
```

只有在函数是 *unqualified-id* 的时候会使用该查找方式。

```cpp
namespace N {
    struct S { };
    void f(S);
}
void g() {
    N::S s;
    f(s); // OK: calls N::f
    (f)(s); // error: N::f not considered; parentheses prevent argument-dependent lookup
}
```

当通常的 unqualified name lookup 可以找到

- 一个类成员
- 一个块级函数声明（除去 *using-declaration*）
- 非函数或函数模板的声明时

时，Argument-dependent name lookup 不生效。否则，查找的结果将包含普通查找结果与 Argument-dependent name lookup 结果的并集。

Argument-dependent name lookup 实际是在参数类型所在的名字空间中查找，并且：

- 忽略 *using-directive*
- 非函数与函数模板的声明将被忽略
- 定义在类中的有元函数可以被找到

## Qualified name lookup

形如 `A::B::C` 形式，对 `B`, `C` 的查找都是 qualified name lookup。

以 `::` 开始的在全局名字空间中查找。

如果 `::` 前的部分为一个枚举，其后应为该枚举中的 enumerator 。

`...` *type-name* `::` `~` *type-name* 中（两个 *type-name* 不一定一致），第二个 *type-name* 与 第一个 *type-name* 在相同的范围内查找。

当 `::` 前不是枚举时，它可能是一个类，或名字空间。

### 类成员

当 `::` 前是一个类时，在类成员及基类中查找。（注意，是基类，不是基类的成员。）但是有如下几个例外：

- 对析构函数查找（见上）
- *conversion-function-id* 中的 *conversion-type-id* 的查找同类成员访问
- *template-id* 中的 *template-argument* 在整个 *postfix-expression* 的上下文中查找
- *using-declaraion* 中的名字，可以找到当前被隐藏类或enumeration

### 名字空间成员

对名字空间成员的查找，会首先名字空间本身的成员。只有当此查找无结果时，才会去查找通过 *using-directive* 引入的成员。

## Elaborated type specifiers

形如 `class A`, `struct B::C` 等。

对于 unqualifed-id ，按照 unqualified name lookup 规则查找，但是忽略一切非类型的声明。

对 `class`, `struct`, `union`，如果找不到则可能声明一个新的类。

如果其为 qualified-id ，那么使用 Qualified name lookup 规则查找，同时忽略一切非类型的名字。此时，即使找不到也不会声明新的类型。

## Class member access

`a.b`, `p->c`

如果 `id-expression` 是一个 unqualified-id，那么在对象类中进行查找。

`a.B::b` 形式中，`B` 将首先在 a 的类中查找。如找不到，则会在整个 *postfix-expression* 上下文中查找。

对 *simple-template-id* 中的 *template-arguments* 在整个 *postfix-expression* 上下文中查找。

在 *conversion-function-id* 中，其 *conversion-type-id* 在对象类中查找。如找不到，则在整个 *postfix-expression* 中查找。所有查找过程中，类型（或类型模板）以外的名字被忽略。
