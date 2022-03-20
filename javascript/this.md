# this

Javascript `this` 的绑定是一个老大难问题。这里顺着标准捋一下 `this` 的问题。

注：标准在不断更新，以下引用与最新标准在语言上可能有所不同。

## 获取 `this` 绑定的对象

解决 `this` 绑定的问题，首先要看一下，当程序里出现 `this` 的时候，到底是如何获取它绑定的对象的呢？

标准里，通过一个叫做 [ResolveThisBinding](https://262.ecma-international.org/#sec-resolvethisbinding) 的内置方法获取 `this` 的绑定，这个方法本身很简单：

> 1. Let *envRec* be GetThisEnvironment().
> 2. Return ? *envRec*.GetThisBinding().

首先通过 [GetThisEnvironment](https://262.ecma-international.org/#sec-getthisenvironment) 拿到保存了 `this` 的环境，然后通过这个环境的 GetThisBinding 内置方法得到 `this`。

### GetThisEnvironment

GetThisEnvironment 就是从当前的环境开始，一级一级向外找，直到找到一个由 `this` 的环境为止：

> 1. Let *env* be the running execution context's LexicalEnvironment.
> 2. Repeat,
>    1. Let *exists* be *envRec*.HasThisBinding().
>    1. If *exists* is **true**, return *env*.
>    1. Let *outer* be the value of *env*.[[OuterEnv]].
>    1. Assert: *outer* is not **null**.
>    1. Set *env* to *outer*.

什么样的环境有 `this` 呢？其实，只有 Function 跟 Global 环境才有 `this` 记录，其他环境，如块（Block），是没有的。

这时，一件神奇的事情发生了。在所有的函数环境里，仅有箭头函数，它的环境里是没有 `this` 记录的。由于 GetThisEnvironment 算法会一直向外找，直到找到有 `this` 记录的环境为止，因而就有了有关 `this` 的第一条规则：**箭头函数会使用包含它的函数（或全局环境）的 `this`。**

### GetThisBinding

接下来，来看 GetThisBinding。对不同的环境，它的定义并不相同。

#### 全局

[全局环境](https://262.ecma-international.org/#sec-global-environment-records-getthisbinding)比较简单，它直接返回了一个 [[GlobalThisValue]] 的槽（可以认为是内置属性）：

> 1. Return *env*.[[GlobalThisValue]].

这个 [[GlobalThisValue]] 又是啥呢？其实这个是由实现决定的。在很多实现里，它就是全局对象（比如浏览器里的 window）。

#### Module 全局

在 [Module 的全局环境](https://262.ecma-international.org/#sec-module-environment-records-getthisbinding) 就更简单了：

> 1. Return **undefined**.

注意即使 `this` 绑定是 `undefined` ，绑定本身也是存在的。检测绑定是否存在，基本通过环境的类型就已经确定了。

#### 函数

[函数环境](https://262.ecma-international.org/#sec-function-environment-records-getthisbinding)就略微复杂一些：

> 1. Assert: *envRec*.[[ThisBindingStatus]] is not **"lexical"**.
> 1. If *envRec*.[[ThisBindingStatus]] is **"uninitialized"**, throw a **ReferenceError** exception.
> 1. Return *envRec*.[[ThisValue]].

其中，检测[[ThisBindingStatus]] 不为 "lexical" 实际是说这不能是一个箭头函数（箭头函数没有 `this` 绑定。

第二部检测如果 `this` 绑定没有被初始化过，那么抛出异常。啥时候初始化的，以后再说（比如，在派生类构造函数里，调用`super(...)`之前）。

所有检测都通过了，那么可以返回环境里记录 `this` 绑定了。

于是除了箭头函数之外，`this` 直接使用了环境里记录的 `this` 绑定。于是函数里的 `this` 是啥，其实就看运行时环境里的 `this` 绑定到了哪里。

## 普通函数调用

除了箭头函数之外，其他函数里的 `this` 是啥，就看环境里的 `this` 绑定到了哪里。
函数环境的 `this` 是通过 [BindThisValue](https://262.ecma-international.org/#sec-bindthisvalue) 来绑定的。

### OrdinaryCallBindThis(F, CalleeContext, thisArgument)

通观标准，只用两个地方引用了这个方法，一个是 [OrdinaryCallBindThis](https://262.ecma-international.org/#sec-bindthisvalue) ，另一个是 `super` 。`super` 用于构造的，我们一会再看。这里先看一下 OrdinaryCallBindThis(F, calleeContext, thisArgument):

> 1. Let *thisMode* be *F*.[[ThisMode]].
> 1. If *thisMode* is **lexical**, return NormalCompletion(`undefined`).
> 1. Let *calleeRealm* be F.[[Realm]].
> 1. Let *localEnv* be the LexicalEnvironment of *calleeContext*.
> 1. If *thisMode* is **strict**, let *thisValue* be *thisArgument*.
> 1. Else,
>    1. If thisArgument is `undefined` or `null`, then
>       1. Let *globalEnv* be *calleeRealm*.[[GlobalEnv]].
>       1. Assert: *globalEnv* is a global Environment Record.
>       1. Let thisValue be *globalEnv*.[[GlobalThisValue]].
>    1. Else,
>       1. Let *thisValue* be ! ToObject(*thisArgument*).
>       1. NOTE: ToObject produces wrapper objects using *calleeRealm*.
> 1. Assert: *localEnv* is a function Environment Record.
> 1. Assert: The next step never returns an abrupt completion because *localEnv*.[ [ThisBindingStatus]] is not **"initialized"**.
> 1. Return *localEnv*.BindThisValue(*thisValue*).

这里 F 是被调用的函数，thisArgument 是待绑定的 `this` 值。

这里有几件事情需要注意：

1. 第 2 步检测了 *thisMode* ，如果为 **lexical**，不做绑定，直接返回。这实际是在检测箭头函数。当前只有箭头函数的 *thisMode* 为 **lexical**。
2. 如果函数定义在严格模式下，thisArgument 将直接作为 `this` 绑定。但是，如果函数定义在非严格模式下，`undefined` 与 `null` 会被替换为全局环境的 `this` ，一般就是全局对象；其他（基本类型）值将被转换为对象。

上面第二点，就是**没有调用对象的时候，`this` 指向全局对象**的来源。

### [[Call]](thisArgument, arumentsList)

使用 OrdinaryBindThis 的，是普通函数对象的 [[Call]] 方法和 [[Construct]] 方法。[[Construct]] 方法用于构造，一会再看。[[[Call]](thisArgument, argumentsList)](https://262.ecma-international.org/#sec-ecmascript-function-objects-call-thisargument-argumentslist) 则是无条件的将传入的 thisArgument 转给了 OrdinaryBindThis 。

### Call(F, V, argumentsList)

调用对象的 [[Call]] 方法的，是内置方法 [Call(F, V, argumentList)](https://262.ecma-international.org/#sec-call) 。它直接使用了 F.[[Call]](V, argumentList) 。

### EvaluateCall(func, ref, arguments, tailPosition)

在函数调用的过程中，使用 [EvaluateCall](https://262.ecma-international.org/#sec-evaluatecall) 方法，其中调用了 Call。

> 1. If *ref* is a Reference Record, then
>    1. If IsPropertyReference(*ref*) is **true**, then
>       1. Let *thisValue* be GetThisValue(*ref*).
>    1. Else,
>       1. Let *refEnv* be *ref*.[[Base]].
>       1. Assert *refEnv* is an Environment Record.
>       1. Let *thisValue* be *refEnv*.WithBaseObject().
> 1. Else,
>    1. Let *thisValue* be `undefined`.
> 1. Let *argList* be ? ArgumentListEvaluation of *arguments*.
> 1. If Type(*func*) is not Object, throw a **TypeError** exception.
> 1. If IsCallable(*func*) is **false**, throw a **TypeError** exception.
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

[Reference](https://262.ecma-international.org/#sec-reference-specification-type) 是一种标准内置类型。它用来表示标识符解析的结果，也就是说，在什么地方找到了某一个标识符。

它一般记录了以下几个信息：

1. base value: 这个标识符是在哪里找到的。它可以一个 Object, 基本类型的值，或者是一个环境（Environment Record），或者是 `undefined`
   1. 对于对象属性，base value 将是包含这个标识符的对象。对象属性访问的形式（[Property Access](https://262.ecma-international.org/#sec-property-accessors)，如 `A.B`， `A["B"]`，以及 `super.Property`）的结果都会是一个Reference，其中 base value 将是其中相当于对象的部分的值。(注意不会检查对象中是否真的存在这个属性)
   2. 对于变量/常量，如局部变量，全局变量，函数参数等，或者说一个单独的 [Identifier](https://262.ecma-international.org/#sec-identifiers)，求值的结果是一个 Reference ，其中的 base value 将是包含这个变量的环境。查找会从标识符出现的环境开始，一层层向上找，直到全局环境。
   3. 变量没有找到的时候，base value 为 `undefined`。（只有单独的 Identifier 没有找到时会有此结果）
2. referenced name: 这是一个字符串。表示标识符的名字。
3. strict: 引用标识符的地点是否处于严格模式

由 `super.Property` 得到 Reference 比较特殊，它一般只用在类成员中，Reference 的 base value 是其父类构造函数的 `prototype` 。同时，它还记录了一个 thisValue，这是其它 Reference 所没有的。这个 `thisValue` ，记录了 `super.Property` 语句所在环境的 `this` 。

Property Access、super.Property 和 Identifier 的求值结果是 Reference 。`(Expr)` 的求值结果与 `Expr` 一致。其它所有表达式的求值结果都不是 Reference 。

EvalueteCall 中用的 [GetThisValue](https://262.ecma-international.org/#sec-getthisvalue) ，会返回 Reference 中记载的 thisValue （如果存在），或者 base value ：

> 1. Assert: IsPropertyReference(V) is true.
> 1. If IsSuperReference(V) is true, then
>    1. Return the value of the thisValue component of the reference V.
> 1. Return GetBase(V).

#### EvaluateCall 中确定 thisValue 的规则

EvaluateCall 中的 *ref* 是对函数调用里函数部分的求值结果。
从 Reference 的介绍，以及 EvaluateCall 的算法，可以得到初始 thisValue 的设置规则：

1. 如果函数名是由一个表达式计算出来的，那么 thisValue 是 `undefined` 。
   * 比如 `(x?func1:func2)()`
   * 此时 *ref* 不是 Reference
1. 如果它是由 Property Access （`A.B`, `A[B]`, `super.B`）生成的 （IsPropertyReference(*ref*) is **true**），那么:
   1. 对 `A.B` 使用 base value，也就是 `A`
   1. 对 `super.B` 使用 thisValue ，也就是 `super.B` 所在函数的 `this`
1. 如果函数名是一个单独的标识符，Reference 的 base value 是一个环境，那么返回这个环境的 WithBaseObject()
   * 此值仅当标识符解析为一个 `with` 块的对象的属性时，为该 `with` 块的对象。其余均为 `undefined` 。

### 小结（函数调用）

对函数调用来说，决定 `this` 经历了以下几个过程：

1. 初始值：(EvaluteCall)
   1. 对 `A.func()`，为 `A`
   2. 对 `super.func()`，为 `super.func()` 语句所在函数的 `this`
   3. 对 `with (obj) { func() ... }` ，如果 `func` 解析为 `obj` 的属性，为 `obj`
   4. 其余均为 `undefined`
2. 非严格模式函数替换 `undefined` ：（OrdinaryCallBindTHis）
   * 非严格模式函数，`undefined` 会被替换为全局环境的 `this`
   * 此处仅检查函数定义是否在严格模式。与调用处是否严格模式无关
3. 非箭头函数，将以上求得的值写入函数运行时环境

读取 `this` 的值时，除了箭头函数，直接从被调用函数的环境中读取。对于箭头函数，从包含箭头函数定义的环境中读取。（注意，不是箭头函数的调用者）（也可以认为，箭头函数在定义时，对外层的 `this` 形成了一个闭包）

## 其它函数调用/系统回调函数

除了直接调用之外，Javascript 函数还可以通过 `call`, `apply` 来调用。这两中调用方式类似，都可以认为是对上面提到的系统内置的 Call 方法的一个封装。

通过这种方式调用，不会经历 `EvaluateCall` ，而是以一个指定的 thisValue 来调用函数。这个 thisValue 会被写进被调用函数的运行时环境。

当然，由于箭头函数运行时环境没有记录 thisValue ，这中方式设置 thisValue 对箭头函数是无效的。

ECMA-262 中规定的很多内置函数会又回调函数，比如 `forEach` 。这些回调函数通常会通过 Call 指定 thisValue 为 `undefined` 调用（同样，对箭头函数不生效）。个别函数（比如 `forEach`）可以调用时通过参数指定调用回调时的 thisValue。

## `bind`

`bind` 会生成一个新的函数对象。这个新的函数对象在生成时记录了调用原函数对象时需要使用的 thisValue 。

当 `bind` 返回的函数对象被调用时，会通过 Call 以记录下的 thisValue 调用被绑定的函数对象。（同将，对箭头函数时无效的。）

## 构造

Javascript 中，使用 `new Func()` 的方式调用构造函数，会使用构造函数的 [[Construct]] 方法来执行。注意一个函数可以同时有 [[Call]] 与 [[Construct]] ，但是两

### [[Construct]](argumentList, newTarget)

与 [[Call]] 不同，[[[Construct]](argumentList, newTarget)](https://262.ecma-international.org/#sec-ecmascript-function-objects-construct-argumentslist-newtarget) 并没有一个参数指明 thisValue 。

`newTarget` 是 `new Func()` 表达式中被调用的构造函数 ，也就是 `Func` 。需要注意的是，即使 `Func` 是一个类（使用 `class` 定义的），并且有基类（在定义时有 `extends Base`），那么在 `Func` 中会使用 `super(...)` 来构造基类，此时会执行基类的构造（Base），但在执行基类的构造时， NewTarget 依然为 `Func` 。也就是说，NewTarget 永远指向 new 表达式中的那个构造函数。

[[[Construct]](argumentList, newTarget)](https://262.ecma-international.org/#sec-ecmascript-function-objects-construct-argumentslist-newtarget) 的执行过程如下（*F* 时构造函数对象）：

> 1. Assert: *F* is an ECMAScript function object.
> 1. Assert: Type(*newTarget*) is Object.
> 1. Let *callerContext* be the running execution context.
> 1. Let *kind* be *F*.[[ConstructorKind]].
> 1. If *kind* is **"base"**, then
>    1. Let *thisArgument* be ? OrdinaryCreateFromConstructor(*newTarget*, **"%ObjectPrototype%"**).
> 1. Let *calleeContext* be PrepareForOrdinaryCall(*F*, *newTarget*).
> 1. Assert: *calleeContext* is now the running execution context.
> 1. If *kind* is **"base"**, perform OrdinaryCallBindThis(*F*, *calleeContext*, *thisArgument*).
> 1. Let *constructorEnv* be the LexicalEnvironment of calleeContext.
> 1. Let *envRec* be constructorEnv's EnvironmentRecord.
> 1. Let *result* be OrdinaryCallEvaluateBody(*F*, *argumentsList*).
> 1. Remove *calleeContext* from the execution context stack and restore *callerContext* as the running execution context.
> 1. If *result*.[[Type]] is return, then
>    1. If Type(*result*.[[Value]]) is Object, return NormalCompletion(*result*.[[Value]]).
>    1. If *kind* is "base", return NormalCompletion(*thisArgument*).
>    1. If *result*.[[Value]] is not `undefined`, throw a `TypeError` exception.
> 1. Else, ReturnIfAbrupt(*result*).
> 1. Return ? *envRec*.GetThisBinding().

注意 *F* 是正在被调用的构造函数，*NewTarget* 是 `new` 表达式中的构造函数。在通过 `super(...)` 指向到基类构造的时候，两这是不同的：*F* 是基类构造函数，*NewTarget* 依然是 `new` 表达式中的派生类构造函数。其余情况下，两者是相同的。

根据第 5 步与第 8 步，仅仅在 *F*.[[ConstructorKind]] 为 **"base"** 的时候，才会创建一个新对象（其原型对象是 *NewTarget*`.prototype`），并将这个新创建的对象通过 OrdinaryCallBindThis 绑定到被调用函数的运行时环境。

其余情况下，在函数开始执行的时候，函数环境的 this 绑定并没有被初始化。（构造函数一定不是箭头函数，其运行时环境中一定存在一个 this 绑定）

[[ConstructorKind]] 现在又两个可能的取值，**"base"** 与 **"derived"** 。仅当构造函数使用 `class` 定义，并且有基类 (`extends Base`) 时（也就是派生类的构造函数），[[ConstructorKind]] 才时 **"derived"** 。其余所有构造函数的 [[ConstructorKind]] 都是 **"base"** （包括所有不使用 `class` 语法定义的构造函数）。为方便起见，下面把所有 [[ConstructorKind]] 为 **"base"** 的构造函数成为基类构造函数，[[ConstructorKind]] 为 **"derived"** 的构造函数成为派生类构造函数。

于是，所有基类构造函数的 `this` ，都是构造函数开始执行之间，创建的一个新对象。但是，派生类构造函数的 `this` ，在构造函数开始执行时，是没有初始化的。此时引用 `this` 会抛出异常。

在以构造方式调用基类构造函数时，如果函数不以 `return` 结束，那么函数的返回就是这个新创建的对象。

### `super(...)`

那么派生类构造函数的 `this` 又是哪里来的呢？是通过 [superCall](https://262.ecma-international.org/#sec-super-keyword-runtime-semantics-evaluation) （`super(...)`）调用基类的构造函数生成的：

> 1. Let *newTarget* be GetNewTarget().
> 1. Assert: Type(*newTarget*) is Object.
> 1. Let *func* be ? GetSuperConstructor().
> 1. Let *argList* be ArgumentListEvaluation of *Arguments*.
> 1. ReturnIfAbrupt(*argList*).
> 1. Let *result* be ? Construct(*func*, *argList*, *newTarget*).
> 1. Let *thisER* be GetThisEnvironment().
> 1. Return ? *thisER*.BindThisValue(*result*).

superCall 调用了基类的构造函数，并将构造函数的返回（新创建的对象）绑定到了当前构造函数的运行时环境中。

派生类构造函数，如果不以 return 结束，那么它的返回就是通过 superCall 新生成，并绑定到 `this` 的新对象。

因而，在派生类构造函数中，在 superCall 之前使用 `this` ，或者返回（不通过 return），都会时运行时错误，因为 `this` 并没有被初始化。

### 小结（构造）

在以构造方式调用函数是，基类构造函数的 `this` ，就是新创建的对象。派生类构造函数开始执行时不能引用 `this` ，必须通过 `super(...)` 调用基类构造函数，生成一个新对象，此后这个新对象将成为 `this` 的值。

## 类属性

最新的标准（现在还是 draft）允许在类里定义属性，而且属性可以有默认值，这个默认值可以是一个函数。当这个默认值是一个箭头函数时，其中 `this` 是啥就有些微妙了。比如这样：

```javascript
class A {
   a = () => {console.log(this.b)}
}
```

这个 `this` 是啥呢？

说来话长，现在也还是 draft 状态，就不引用了。简单总结以下：

箭头函数不是直接在类定义里定义的。类里所有的 ` a = b ` 的是带有默认值的属性，默认值部分都会被改写成一个类似于 `function(){ return b }` 的函数。上面的箭头函数也是一样： `function(){ return ()=>{console.log(this.b)}}`。所以，其中的 `this` ，是这个被改写的函数的 `this` 。当这个函数会在构造的过程中被调用，其 `this` 就是构造函数中的 `this` ，通常就是被构造的对象。
