# this

Javascript `this` 的绑定是一个老大难问题。这里顺着标准捋一下 `this` 的问题。

## 获取 `this` 绑定的对象

解决 `this` 绑定的问题，首先要看一下，当程序里出现 `this` 的时候，到底是如何获取它绑定的对象的呢？

标准里，通过一个叫做 [ResolveThisBinding](https://www.ecma-international.org/ecma-262/#sec-resolvethisbinding) 的内置方法获取 `this` 的绑定，这个方法本身很简单：

> 1. Let envRec be GetThisEnvironment().
> 2. Return ? envRec.GetThisBinding().

首先通过 [GetThisEnvironment](https://www.ecma-international.org/ecma-262/#sec-getthisenvironment) 拿到保存了 `this` 的环境，然后通过这个环境的 GetThisBinding 内置方法得到 `this`。

### GetThisEnvironment

GetThisEnvironment 就是从当前的环境开始，一级一级向外找，直到找到一个由 `this` 的环境为止：

> 1. Let lex be the running execution context's LexicalEnvironment.
> 2. Repeat,
>    1. Let envRec be lex's EnvironmentRecord.
>    1. Let exists be envRec.HasThisBinding().
>    1. If exists is true, return envRec.
>    1. Let outer be the value of lex's outer environment reference.
>    1. Assert: outer is not null.
>    1. Set lex to outer.

什么样的环境有 `this` 呢？其实，只有 Function 跟 Global 环境才有 `this` 记录，其他环境，如块（Block），是没有的。

这时，一件神奇的事情发生了。在所有的函数环境里，仅有箭头函数，它的环境里是没有 `this` 记录的。由于 GetThisEnvironment 算法会一直向外找，直到找到有 `this` 记录的环境为止，因而就有了有关 `this` 的第一条规则：**箭头函数会使用包含它的函数（或全局环境）的 `this`。**

### GetThisBinding

接下来，来看 GetThisBinding。对不同的环境，它的定义并不相同。

#### 全局
[全局环境](https://www.ecma-international.org/ecma-262/#sec-global-environment-records-getthisbinding)比较简单，它直接返回了一个 [[GlobalThisValue]] 的槽（可以认为是内置属性）：

> 1. Let *envRec* be the global Environment Record for which the method was invoked.
> 1. Return *envRec*.[[GlobalThisValue]].

这个 [[GlobalThisValue]] 又是啥呢？其实这个是由实现决定的。在很多实现里，它就是全局对象。

#### Module 全局

在 [Module 的全局环境](https://www.ecma-international.org/ecma-262/#sec-module-environment-records-getthisbinding) 就更简单了：

> 1. Return **undefined**.

注意即使 `this` 绑定是 `undefined` ，绑定本身也是存在的。检测绑定是否存在，基本通过环境的类型就已经确定了。

#### 函数

[函数环境](https://www.ecma-international.org/ecma-262/#sec-function-environment-records-getthisbinding)就略微复杂一些：

> 1. Let *envRec* be the function Environment Record for which the method was invoked.
> 1. Assert: *envRec*.[[ThisBindingStatus]] is not **"lexical"**.
> 1. If *envRec*.[[ThisBindingStatus]] is **"uninitialized"**, throw a **ReferenceError** exception.
> 1. Return *envRec*.[[ThisValue]].

其中第二步，[[ThisBindingStatus]] 不为 "lexical" 实际是说这不能是一个箭头函数（箭头函数没有 `this` 绑定。

第三部检测如果 `this` 绑定没有被初始化过，那么抛出异常。啥时候初始化的，以后再说。

所有检测都通过了，那么可以返回环境里记录 `this` 绑定了。

于是除了箭头函数之外，`this` 直接使用了环境里记录的 `this` 绑定。于是函数里的 `this` 是啥，其实就看运行时环境里的 `this` 绑定到了哪里。
