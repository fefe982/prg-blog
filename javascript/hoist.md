# Javascript 中的变量提升

## 为什么要提升

变量提升是 Javascript 中一个很有趣，也让很多人迷惑的特征。那么，Javascript 为什么要设计这个特征呢？

我来看 Javascript 创始人 Brendan Eich 的 [twitter](https://twitter.com/BrendanEich/status/562313394431078400):

> A bit more history: `var` hoisting was an implementation artifact. `function` hoisting was better motivated: <https://twitter.com/BrendanEich/status/33403701100154880>.

函数的提升使用明确的理由的。但是变量的提升，只是再实现的“顺便”提升了。

那么函数为什么要提升呢？[他也给出了理由](https://twitter.com/BrendanEich/status/33403701100154880)：

> yes, function declaration hoisting is for mutual recursion & generally to avoid painful bottom-up ML-like order

函数提升是为了可以再函数定义之前调用函数。只有这样才可能支持两个函数之间互相调用。同时，这样可以把程序的主要逻辑放在前部，而不是必须放在程序最后，是程序结果更加符合人的书写与阅读习惯。

## 如何提升

很多介绍变量提升的文章，提到变量提升可以这样理解：

```javascript
/*... 一些代码 ...*/
var x = 1;
```

等价于

```javascript
var x;
/*... 一些代码 ...*/
x = 1;
```

这很直观。但是 Javascript 编译、运行过程中，显然不会随便修改用户的代码。那么，变量提升在 Javascript 中具体是如何实现的呢？

### Javascript 的运行过程

为了解决这个问题，我们先看一下 Javascript 的整理运行逻辑。

在 ECMA 262 里，对 [ScriptEvaluationJob](https://www.ecma-international.org/ecma-262/#sec-scriptevaluationjob) 如瑞啊的描述：

1. Assert: sourceText is an ECMAScript source text (see clause 10).
2. Let realm be the current Realm Record.
3. Let s be ParseScript(sourceText, realm, hostDefined).
4. If s is a List of errors, then
   1. Perform HostReportErrors(s).
   2. Return NormalCompletion(undefined).
5. Return ? ScriptEvaluation(s).

对 Module 来说，有 [TopLevelModuleEvaluationJob](https://www.ecma-international.org/ecma-262/#sec-toplevelmoduleevaluationjob):

1. Assert: sourceText is an ECMAScript source text (see clause 10).
2. Let realm be the current Realm Record.
3. Let m be ParseModule(sourceText, realm, hostDefined).
4. If m is a List of errors, then
   1. Perform HostReportErrors(m).
   2. Return NormalCompletion(undefined).
5. Perform ? m.Instantiate().
6. Assert: All dependencies of m have been transitively resolved and m is ready for evaluation.
7. Return ? m.Evaluate().

可以看到，Javascirpt 虽然是解释性执行的语言，但是它并不是边读取边解释边执行，而是一定要把整个脚本加载并解析完成（通过 `ParseScript` 或 `ParseModule`）之后，才开始执行。这样，在脚本开始执行的时候，就可以知道所有的变量与函数的声明的信息，即使还没有执行到变量或函数声明的地点。这就使得在 Javascript 里引用“还没有声明”的函数和变量成为可能。

### 不同作用域的变量提升

变量提升会在函数内，以及全局作用域发生。

#### 全局（*Script*)

在 ECMA-262 中，通过 **VarScopedDeclarations** 来收集 *Script* 中的 var 变量定义，以及顶级函数定义，并在执行脚本之前，通过 [GlobalDeclarationInstantiation](https://www.ecma-international.org/ecma-262/#sec-globaldeclarationinstantiation) 注册至全局环境。这些变量以及函数将被放在全局对象中。变量在此时将被初始化为 `undefined` ，而函数则是直接被初始化为函数本身（可以直接调用）。

*Script* 全局的 **VarScopedDeclarations** 将收集：

1. `var` 声明的变量。包括 `for(var ...)`、`for await (var ...)` 中用 var 声明的变量。同时，在各种控制结构内部，以及 Block 内部的都会被一起收集。这一过程实在运行之前执行的，所以声明是否会提升与代码是否会被执行无关。如 `if` 等控制语句中，未被执行的代码中定义的变量也会被提升。但是，函数定义内部的 `var` 定义不会被收集，也就是说函数内部的 `var` 定义不会被提升至函数外。
2. 顶级的函数声明。位于块内部，以及各种控制结构内部的函数声明不会被收集，也不会被提升。

说点细节的话，*Script* 的 **VarScopedDeclarations** 直接使用了其中的 *StatementList* 的 **TopLevelVarScopedDeclarations** 。*StatementList* 可以包含 *Statement* 与 *Declaration* 。*StatementList* 的 **TopLevelVarScopedDeclarations** 收集了 *Statement* 的 **VarScopedDeclarations** 与 *Declaration* 中的函数声明。

*Statement* 的 **VarScopedDeclaration** 与 *Script* 不同，直接递归收集了语句中（除函数定义体内部之外的）所有 `var` 声明。

因而对于函数声明，仅有顶级的会被提升。

#### 全局（*Module*)

*Module* 的做法略有不同。它通过 **VarScopedDeclarations** 来收集 var 变量定义，通过 **LexicallyScopedDeclarations** 来收集顶级函数定义，并在执行脚本之前，使用 [InitializeEnvironment](https://www.ecma-international.org/ecma-262/#sec-source-text-module-record-initialize-environment) 注册至全局环境。*Module* 没有全局对象，全局变量是被放在全局的 *Module* 的 Lexical Scope 里的。不同的 *Module* 之间，不会互相影响。变量同样会被初始化为 `undefined`，而函数可以直接使用。

#### 函数中

函数中的情形与 *Script* 类似。只不过它不通过 *Script* ，而是使用 *FunctionBody* 的 **VarScopedDeclarations** 来收集需要提升的定义，并在 [FunctionDeclarationInstantiation](https://www.ecma-international.org/ecma-262/#sec-functiondeclarationinstantiation) 里注册到运行环境中。

但是，函数中的 var 变量定义并不保存在对象里。且 `var` 与 函数参数可以认为是出来同一个环境中的，所以，对于与函数参数同名的 `var` 变量，它的初始值将是函数的参数值，其它依然为 `undefined` 。

#### 再说函数定义的提升

上面说了，`var` 是可以跨块提升的，但是函数不可以。除了顶级（全局的最外层，或函数定义的最外层）的函数意外，其他的函数声明将通过 **LexicallyScopedDeclarations** ，并在进入块的时候，通过 [BlockDeclarationInstantiation](https://www.ecma-international.org/ecma-262/#sec-blockdeclarationinstantiation) 定义在一个新建的块级作用域中。

但是，在 ECMAScript 将块级函数标准化之前，各家浏览器就已经各自实现了块内定义的函数。这就导致大家的实现各不相同，并且持续至今。这也包括块内声明的函数是如何提升的。因而，可以在浏览器里观察到与此处描述不同的块级定义的函数的提升行为。[MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/function) 有不同浏览器块内函数提升的在不同浏览器中的对比。

#### `let` 与 `const`

`let` 与 `const` 一般被认为不提升。但是这也不太准确。他们与 `var` 有两点不同。

1. `let` 与 `const` 不会跨块。他们定义的变量仅在块内（以及快内的嵌套块、函数内）可以访问。在块外不存在。（`var` 会提升到函数的顶层。）这与函数是一致的。并且与函数一样，它的定义是通过 **LexicallyScopedDeclarations** 收集的。
2. 在 *Script* 全局，它们也不会被放进全局对象，而是一定处在一个 [Lexical Enviroment](https://www.ecma-international.org/ecma-262/#sec-lexical-environments) 中。
3. 他们在创建的时候，不会被初始化。而是一定要执行到定义的语句，才会被初始化。(`var` 变量在创建时，会立即被初始化为 `undefined`。)

在 Javascript ，使用未被初始化的变量会抛出异常。所以 `var` 提升之后，可以在定义之前使用（因为被初始化了），但是 `let` 与 `const` 在作用域之内，定义之前使用就会抛出异常（因为他们进入作用域就已经存在了，但是未被初始化）。但是，如果在作用域外使用，在非严格模式下，会导致在全局对象中创建一个同名变量（因为他们不存在！），反而不会出错。

#### node.js

这个为啥要拿出来说呢，因为 node.js 的（除 [ECMAScript module](https://nodejs.org/api/esm.html) 外）代码，[都是会被放进一个函数运行的](https://nodejs.org/api/modules.html#modules_the_module_wrapper)：

```javascript
(function(exports, require, module, __filename, __dirname) {
// Module code actually lives in here
});
```

所以，在 node.js 里，无法在全局作用域写代码。因而 `var` 声明的“全局”变量，也不会进入全局对象。
