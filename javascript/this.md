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

## 绑定 `this` （Call）

除了箭头函数之外，其他函数里的 `this` 是啥，就看环境里的 `this` 绑定到了哪里。
函数环境的 `this` 是通过 [BindThisValue](https://www.ecma-international.org/ecma-262/#sec-bindthisvalue) 来绑定的。

### OrdinaryCallBindThis(F, CalleeContext, thisArgument)

通观标准，只用两个地方引用了这个方法，一个是 [OrdinaryCallBindThis](https://www.ecma-international.org/ecma-262/#sec-bindthisvalue) ，另一个是 `super` 。`super` 用与构造的，我们一会再看。这里先看一下 OrdinaryCallBindThis(F, calleeContext, thisArgument):

> 1. Let *thisMode* be F.[[ThisMode]].
> 1. If *thisMode* is **lexical**, return NormalCompletion(`undefined`).
> 1. Let *calleeRealm* be F.[[Realm]].
> 1. Let *localEnv* be the LexicalEnvironment of *calleeContext*.
> 1. If *thisMode* is **strict**, let *thisValue* be *thisArgument*.
> 1. Else,
>    1. If thisArgument is `undefined` or `null`, then
>       1. Let *globalEnv* be *calleeRealm*.[[GlobalEnv]].
>       1. Let *globalEnvRec* be *globalEnv*'s EnvironmentRecord.
>       1. Assert: *globalEnvRec* is a global Environment Record.
>       1. Let thisValue be *globalEnvRec*.[[GlobalThisValue]].
>    1. Else,
>       1. Let *thisValue* be ! ToObject(*thisArgument*).
>       1. NOTE: ToObject produces wrapper objects using *calleeRealm*.
> 1. Let *envRec* be *localEnv*'s EnvironmentRecord.
> 1. Assert: *envRec* is a function Environment Record.
> 1. Assert: The next step never returns an abrupt completion because *envRec*.[ [ThisBindingStatus]] is not **"initialized"**.
> 1. Return *envRec*.BindThisValue(*thisValue*).

这里 F 是被调用的函数，thisArgument 是待绑定的 `this` 值。

这里有几件事情需要注意：

1. 第 2 步检测了 *thisMode* ，如果为 **lexical**，不做绑定，直接返回。这实际是在检测箭头函数。当前只有箭头函数的 *thisMode* 为 **lexical**。
2. 如果函数定义在严格模式下，thisArgument 将直接作为 `this` 绑定。但是，如果函数定义在非严格模式下，`undefined` 与 `null` 会被替换为全局环境的 `this` ，一般就是全局对象；其他（基本类型）值将被转换为对象。

上面第二点，就是**没有调用对象的时候，`this` 指向全局对象**的来源。

### [[Call]](thisArgument, arumentsList)

使用 OrdinaryBindThis 的，是普通函数对象的 [[Call]] 方法和 [[Construct]] 方法。[[Construct]] 方法用于函数，一会再看。[[[Call]](thisArgument, argumentsList)](https://www.ecma-international.org/ecma-262/#sec-ecmascript-function-objects-call-thisargument-argumentslist) 则是无条件的将传入的 thisArgument 转给了 OrdinaryBindThis 。

### Call(F, V, argumentsList)

调用对象的 [[Call]] 方法的，是内置方法 [Call(F, V, argumentList)](https://www.ecma-international.org/ecma-262/#sec-call) 。它直接使用了 F.[[Call]](V, argumentList) 。

### EvaluateCall(func, ref, arguments, tailPosition)

在函数调用的过程中，使用 [EvaluateCall](https://www.ecma-international.org/ecma-262/#sec-evaluatecall) 方法，其中调用了 Call。

> 1. If Type(*ref*) is Reference, then
>    1. If IsPropertyReference(*ref*) is **true**, then
>       1. Let *thisValue* be GetThisValue(*ref*).
>    1. Else the base of *ref* is an Environment Record,
>       1. Let *refEnv* be GetBase(*ref*).
>       1. Let *thisValue* be *refEnv*.WithBaseObject().
> 1. Else Type(*ref*) is not Reference,
>    1. Let *thisValue* be `undefined`.
> 1. Let *argList* be ArgumentListEvaluation of *arguments*.
> 1. ReturnIfAbrupt(*argList*).
> 1. If Type(*func*) is not Object, throw a TypeError exception.
> 1. If IsCallable(*func*) is **false**, throw a TypeError exception.
> 1. If *tailPosition* is **true**, perform PrepareForTailCall().
> 1. Let *result* be Call(*func*, *thisValue*, *argList*).
> 1. Assert: If *tailPosition* is **true**, the above call will not return here, but instead evaluation will continue as if the following return has already occurred.
> 1. Assert: If *result* is not an abrupt completion, then Type(*result*) is an ECMAScript language type.
> 1. Return *result*.

这里，终于出现确定 *thisValue* 的逻辑，但是它与 *ref* 是否是 Reference 有关。*ref* 是啥呢，我们看一下使用 `EvaluateCall` 的地方（有几处，都差不多，这里选了一个简单的）：

> *CallExpression* : *CallExpression* *Arguments*
>
> 1. Let *ref* be the result of evaluating *CallExpression*.
> 1. Let *func* be ? GetValue(*ref*).
> 1. Let *thisCall* be this *CallExpression*.
> 1. Let *tailCall* be IsInTailPosition(*thisCall*).
> 1. Return ? EvaluateCall(*func*, *ref*, *Arguments*, *tailCall*).

*ref* 是函数调用，函数名部分（函数其实可以是一个表达式的结果）的计算结果。*func* 是从 *ref* 中取出的值，也就是被调用的函数。而 *ref* 不一定是一个值，可能是 Reference （这个不是大家常说的引用，而是一种 ECMA-262 内置类型）。GetValue 可以从 Reference 中取出记录的值。

#### Reference

[Reference](https://www.ecma-international.org/ecma-262/#sec-reference-specification-type) 是一种标准内置类型。它用来表示标识符解析的结果，也就是说，在什么地方找到了某一个变量。

它一般记录了以下几个信息：

1. base value: 这个标识符是在哪里找到的。它可以一个 Object, 基本类型的值，或者是一个环境（Environment Record），或者是 `undefined`
   1. 对于对象属性，base value 将是包含这个标识符的对象。对象属性访问的形式（[Property Access](https://www.ecma-international.org/ecma-262/#sec-property-accessors)，如 `A.B`， `A["B"]`，以及 `super.Property`）的结果都会是一个Reference，其中 base value 将是其中相当于对象的部分的值。(注意不会检查对象中是否真的存在这个属性)
   2. 对于变量/常量，如局部变量，全局变量，函数参数等，或者说一个单独的 [Identifier](https://www.ecma-international.org/ecma-262/#sec-identifiers)，求值的结果是一个 Reference ，其中的 base value 将是包含这个变量的环境。查找会从标识符出现的环境开始，一层层向上找，直到全局环境。
   3. 变量没有找到的时候，base value 为 `undefined`。（只有单独的 Identifier 没有找到是会有此结果）
2. referenced name: 这是一个字符串。表示标识符的名字。
3. strict: 引用表示符的地点是否处于严格模式

由 super.Property 得到 Reference 比较特殊，不过它只能用于构造函数里（而且该函数不能被当成普通函数调用），我们在后续讲到构造的时候再讨论。

Property Access、super.Property 和 Identifier 的求值结果是 Reference 。`(Expr)` 的求值结果与 `Expr` 一致。其它所有表达式的求值结果都不是 Reference 。

#### thisValue 的规则

EvaluateCall 中的 *ref* 是对函数调用里函数部分的求值结果。
从 Reference 的介绍，以及 EvaluateCall 的算法，可以得到初始 thisValue 的设置规则：

1. 如果函数名是由一个表达式计算出来的，那么 thisValue 是 `undefined` 。
   * 比如 `(x?func1:func2)()`
   * 此时 *ref* 不是 Reference
1. 如果它是由 Property Access （`A.B`, `A[B]`）生成的 （IsPropertyReference(*ref*) is **true**），那么直接使用 base value 。
   * 暂不考虑 `super.` 的情况
   * GetThisValue(*ref*) 此时直接返回 base value ，也就是 `A`
1. 如果函数名是一个单独的标识符，Reference 的 base value 是一个环境，那么返回这个环境的 WithBaseObject()
   * 此值仅当标识符解析为一个 `with` 块的对象的属性时，为给 `with` 块的对象。其余均为 `undefined` 。

### 小结

对函数调用来说，决定 `this` 经历了以下几个过程：

1. 初始值：(EvaluteCall)
   1. 对 `A.B`，`A[B]`，为 `A`
   1. 对 `with (obj) { property ...}` ，如果 `property` 解析为 `obj` 的属性，为 `obj`
   1. 其余均为 `undefined`
2. 非严格模式函数替换 `undefined` ：（OrdinaryCallBindTHis）
   * 非严格模式函数，`undefined` 会被替换为全局环境的 `this`
   * 此处仅检查函数定义是否在严格模式。与调用处是否严格模式无关
3. 非箭头函数，将以上求得的值写入函数运行时环境

读取 `this` 的值时，除了箭头函数，直接从当前函数的环境中读取。对于箭头函数，从包含箭头函数定义的环境中读取。（注意，不是箭头函数的调用者）（也可以认为，箭头函数在定义时，对外层的 `this` 形成了一个闭包）
