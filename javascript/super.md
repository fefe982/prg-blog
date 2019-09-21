# super

`super` 是一个 javascript 得关键字。也正是由于它是一个关键字，使得标准有机会为它得每一种用法单独定义行为，从而使它得用法与众不同。ECMA262 有专门的一节 [The super keyword](https://www.ecma-international.org/ecma-262/#sec-super-keyword) 来介绍 super 的用法。

## superCall

在派生类（`class Derived extends Base`）的构造里，可以通过 `super(...)` 的方式调用基类的构造函数。这个称作 [superCall](https://www.ecma-international.org/ecma-262/#prod-SuperCall)。在调用 superCall 之前，`this` 绑定并未初始化，不能使用 `this` 。superCall 会使用基类构造函数的“返回值”初始化当前的 `this` 。（这里的“返回值”，类似于 `new Base(...)` 的结果）

## superProperty

在派生类的构造，或者成员函数中，可以使用 `super[prop]` 或者 `super.property` 形式，称作 [superProperty](https://www.ecma-international.org/ecma-262/#prod-SuperProperty) 。

superProperty 的行为跟其他的 `A.B` 的形式的结构表现很对相同。主要的 `super.property` ，`super` 不是一个对象，这不是一个取对象属性的表达。标准直接定义了 `super.property` 作为一个整体的含义。

### Reference

在继续说 superProperty 之前，先要介绍一下 [Reference](https://www.ecma-international.org/ecma-262/#sec-reference-specification-type) ，因为 `A.B` 或 superProperty 的结果都是一个 Reference 。Reference 是一种标准内置类型。它用来表示标识符解析的结果，也就是说，在什么地方找到了某一个标识符。在需要取值的时候或赋值的时候，会通过 Reference 进行。

下面主要介绍 `Reference` 中结构中与，与 `A.B` 与 superProperty 相关的部分。

`A.B` 、superProperty 生成的 Reference 一般记录了以下几个信息：

1. base value: 这个标识符是在哪里找到的。它可以一个 Object, 基本类型的值，或者是一个环境（Environment Record），或者是 `undefined`
   * 在 `A.B` 中，为 `A`。即将会从 `A` 中查找属性 `"B"`。(注意此时不会检查对象中是否真的存在这个属性)
   * 在 `super.B` 中，base value 为 Object.getPrototypeOf([[HomeObject]])（[[HomeObject]]的原型对象。[[HomeObject]]是啥一会介绍）
2. referenced name: 这是一个字符串。表示标识符的名字。
   * 在 `A.B` 中，为 `"B"`
3. strict: 引用标识符的地点是否处于严格模式
4. thisValue: 仅在 `super.B` 得到的 Reference 中存在，为 `super.B` 所在环境的 `this` 。

### [[HomeObject]]

[[HomeObject]] 是专门为 superProperty 存在的。Object.getPrototypeOF([[HomeObject]]) 将成为 superProperty 结果 Reference 的 base value。

只有成员函数（包括 getter、setter）、contructor 才有 [[HomeObject]] ，其它函数没有 [[HomeObject]] 。除箭头函数外，只有在有 [[HomeObject]] 的函数中，才可以使用 superProperty。对箭头函数来说，如果箭头函数定义所在的最内层非箭头函数有 [[HomeObject]] ，那么在箭头函数中也可以使用 superProperty ，他们共享同一个 [[HomeObject]]。（也就是说，直接定义在类成员方法中的箭头函数中可以使用 superProperty。）

#### 类

`class` 的定义是通过 [ClassDefinitionEvaluation](https://www.ecma-international.org/ecma-262/#sec-runtime-semantics-classdefinitionevaluation) 算法处理的。它为 class 的每一个成员函数（包括 getter 、setter）定义了 [[HomeObject]] 。

对于 `class Derived extends Base` ，非 `static` 方法的 [[HomeObject]] 是 `Derived.prototype` ；`static` 方法的 [[HomeObject]] 是 `Derived` 。

根据类的定义，`Object.getPrototypeOf(Derived.prototype) === Base.protorype` 。当 `extends Base` 不存在时（基类定义），`Object.getPrototypeOf(Derived.prototype) === Object.prototype`。也就是说，非 `static` 方法的 [[HomeObject]] 时当前类的原型对象，superProperty 的 base value 时其基类的原型对象。

`Derived` 本身时一个构造函数，根据类的定义，`Object.getPrototypeOf(Derived) === Base` 。当 `extends Base` 不存在时，`Object.getPrototypeOf(Derived) == Function.prototype` 。对于 `static` 方法，[[HomeObject]] 是当前类的构造函数，superProperty 的 base value 是基类的构造函数；没有基类时，是 `Function.prototype` 。

#### object literal

javascript 中的 [Object literal](https://www.ecma-international.org/ecma-262/#sec-object-initializer) 中以 [MethodDefinition](https://www.ecma-international.org/ecma-262/#prod-MethodDefinition) 形式直接定义的方法，也有 [[HomeObject]] 的定义。如下例中，`method` 中就有 [[HomeObject]] 定义。但是， `non_method` 对应的 function 并不是直接定义的，而是作为 `non_method` 属性的值，因而并没有 [[HomeObject]] 定义。

```javascript
{
    method(){}
    non_method : function(){}
}
```

MethodDefinition 的形式包括：

```javascript
{
    method(...){...}
    *generatorMethod(...){...}
    async asyncMethod(...){...}
    async *asyncGeneratorMethod(...){...}
    get propertyName(){...}
    set propertyName(...){...}
}
```

在 object literal 的[构建算法](https://www.ecma-international.org/ecma-262/#sec-object-initializer-runtime-semantics-evaluation)，为成员方法指定了 [[HomeObject]] 。这个 [[HomeObject]] 就是这个 object literal 本身。其原型对象（也就是 superProperty 的 base value）是 `Object.prototype` 。

### 取值

Reference 只记录了应该从何处去找到一个对象，并没有记录对象的值。取值的时候，需要到这个地方把只拿出来。这个操作，在 ECMA262 中称为 [GetValue(V)](https://www.ecma-international.org/ecma-262/#sec-getvalue) （其中 *V* 是需要被取值的 Reference）：

> 1. ReturnIfAbrupt(*V*).
> 2. If Type(*V*) is not Reference, return *V*.
> 3. Let *base* be GetBase(*V*).
> 4. If IsUnresolvableReference(*V*) is `true`, throw a `ReferenceError` exception.
> 5. If IsPropertyReference(*V*) is `true`, then
>    1. If HasPrimitiveBase(*V*) is `true`, then
>       1. Assert: In this case, *base* will never be `undefined` or `null`.
>       2. Set *base* to ! ToObject(*base*).
>    2. Return ? *base*.[[Get]](GetReferencedName(*V*), GetThisValue(*V*)).
> 6. Else *base* must be an Environment Record,
>    1. Return ? *base*.GetBindingValue(GetReferencedName(*V*), IsStrictReference(*V*)) (see 8.1.1).

与本文相关的是第 5 步，最终取值的结果是 *base*.[[Get]](GetReferencedName(*V*), GetThisValue(*V*)) 。

这个 *base* 就是 base value 。[[Get]] 是取对象属性得内置方法。GetReferencedName(*V*) 返回 *V* 中记录得属性名 (referenced name)。GetThisValue(*V*) 返回 *V* 中记录得 thisValue (superProperty)；如果 thisValue 不存在（不是 superProperty），GetThisValue(*V*) 将返回 *V* 中得 baseValue 。

这里，superProperty 与普通的取对象属性并不相同，区别就在于 [[Get]] 的第二个参数。

#### [[Get]]

[[Get]] 是 javascript 对象的一个内置槽，不同的对象会不太相同。对普通的对象来说，它是如下定义的:

*O*.[\[\[Get\]\](*P*, *Receiver*)](https://www.ecma-international.org/ecma-262/#sec-ordinary-object-internal-methods-and-internal-slots-get-p-receiver) ：

> 1. Return ? OrdinaryGet(*O*, *P*, *Receiver*).

[OrdinaryGet(*O*, *P*, *Receiver*)](https://www.ecma-international.org/ecma-262/#sec-ordinaryget) ：

> 1. Assert: IsPropertyKey(*P*) is `true`.
> 2. Let *desc* be ? *O*.\[[GetOwnProperty]](*P*).
> 3. If *desc* is `undefined`, then
>    1. Let *parent* be ? *O*.\[[GetPrototypeOf]]().
>    2. If *parent* is `null`, return `undefined`.
>    3. Return ? *parent*.[[Get]](*P*, *Receiver*).
> 4. If IsDataDescriptor(*desc*) is `true`, return *desc*.[[Value]].
> 5. Assert: IsAccessorDescriptor(*desc*) is `true`.
> 6. Let *getter* be *desc*.[[Get]].
> 7. If *getter* is `undefined`, return `undefined`.
> 8. Return ? Call(*getter*, *Receiver*).

对比 GetValue ，可以看到，OrdinaryGet 中的 *O* 与 *Receiver* ，对普通对象属性，均为 base value ；对 superProperty ，分别是 base value 与 thisValue 。

这里仅有第 8 步用到了 *Receiver* 。也就是说，如果这个属性是一个 getter ，那么 getter 函数中的 `this` 对 superProperty 来说将是 thisValue ，而在普通对象属性中，将是 base value 。

也就是说，在 superProperty 中， base value 提供了属性的定义，thisValue 提供了具体实现。这个定义如果是一个值，那么就直接返回了。这个定义如果是一个 getter ，那么这个函数在 thisValue （具体实现）上执行。

在普通属性取值中，在经过 OrdinaryGet 第三步顺着原型链向上走，并递归调用到 OrdianryGet 时，*O* 与 *Receiver* 也不再相同。*O* 此时是原型对象，*Receiver* 依然是原始取属性的对象。这同样可以理解为原型对象提供了属性定义，而实际对象提供的属性的实现。

### 赋值

向 Reference *V* 赋值 *W* ，是通过 [PutValue(*V*, *W*)](https://www.ecma-international.org/ecma-262/#sec-putvalue) 实现的：

> 1. ReturnIfAbrupt(*V*).
> 2. ReturnIfAbrupt(*W*).
> 3. If Type(*V*) is not Reference, throw a `ReferenceError` exception.
> 4. Let *base* be GetBase(*V*).
> 5. If IsUnresolvableReference(*V*) is `true`, then
>    1. If IsStrictReference(*V*) is `true`, then
>       1. Throw a `ReferenceError` exception.
>    2. Let *globalObj* be GetGlobalObject().
>    3. Return ? Set(*globalObj*, GetReferencedName(*V*), *W*, `false`).
> 6. Else if IsPropertyReference(*V*) is `true`, then
>     1. If HasPrimitiveBase(*V*) is `true`, then
>         1. Assert: In this case, *base* will never be `undefined` or `null`.
>         2. Set *base* to ! ToObject(*base*).
>     2. Let *succeeded* be ? *base*.\[[Set]](GetReferencedName(*V*), *W*, GetThisValue(*V*)).
>     3. If *succeeded* is `false` and IsStrictReference(*V*) is `true`, throw a `TypeError` exception.
>     4. Return.
> 7. Else *base* must be an Environment Record,
>     1. Return ? *base*.SetMutableBinding(GetReferencedName(*V*), *W*, IsStrictReference(*V*)) (see 8.1.1).

与本文有关的是第 6 步，它使用了 base value 的 [[Set]] 方法来设置属性值。

对普通对象来说，它是如下定义的：

*O*.[\[\[Set\]\](*P*, *V*, *Receiver*)](https://www.ecma-international.org/ecma-262/#sec-ordinary-object-internal-methods-and-internal-slots-set-p-v-receiver) ：

> 1. Return ? OrdinarySet(*O*, *P*, *V*, *Receiver*).

[OrdinarySet(*O*, *P*, *V*, *Receiver*)](https://www.ecma-international.org/ecma-262/#sec-ordinaryset) ：

> 1. Assert: IsPropertyKey(P) is true.
> 1. Let ownDesc be ? O.[[GetOwnProperty]](P).
> 1. Return OrdinarySetWithOwnDescriptor(O, P, V, Receiver, ownDesc).

[OrdinarySetWithOwnDescriptor(O, P, V, Receiver, ownDesc)](https://www.ecma-international.org/ecma-262/#sec-ordinarysetwithowndescriptor) ：

> 1. Assert: IsPropertyKey(*P*) is true.
> 2. If *ownDesc* is `undefined`, then
>    1. Let *parent* be ? *O*.\[\[GetPrototypeOf]]().
>    2. If *parent* is not `null`, then
>       1. Return ? *parent*.\[\[Set]](*P*, *V*, *Receiver*).
>    3. Else,
>       1. Set *ownDesc* to the PropertyDescriptor { [[Value]]: `undefined`, [[Writable]]: `true`, [[Enumerable]]: `true`, [[Configurable]]: `true` }.
> 3. If IsDataDescriptor(*ownDesc*) is `true`, then
>    1. If *ownDesc*.[[Writable]] is `false`, return `false`.
>    2. If Type(*Receiver*) is not Object, return `false`.
>    3. Let *existingDescriptor* be ? *Receiver*.[[GetOwnProperty]](*P*).
>    4. If *existingDescriptor* is not `undefined`, then
>       1. If IsAccessorDescriptor(*existingDescriptor*) is `true`, return `false`.
>       2. If *existingDescriptor*.[[Writable]] is `false`, return `false`.
>       3. Let *valueDesc* be the PropertyDescriptor { [[Value]]: *V* }.
>       4. Return ? *Receiver*.[[DefineOwnProperty]](*P*, *valueDesc*).
>    5. Else *Receiver* does not currently have a property *P*,
>       1. Return ? CreateDataProperty(*Receiver*, *P*, *V*).
> 4. Assert: IsAccessorDescriptor(*ownDesc*) is true.
> 5. Let *setter* be *ownDesc*.[[Set]].
> 6. If *setter* is `undefined`, return `false`.
> 7. Perform ? Call(*setter*, *Receiver*, « *V* »).
> 8. Return `true`.

这，可以看到，*O* 依然只是提供了属性的定义。如果 *O* 中的属性不可写，哪个赋值将失败（3.1）。但是，实际写入时发生了 *Receiver* （实现）中的（3.4.4；3.5.1）。如果实现（*Receiver*）中已经存在了这个属性，那么 *Receiver* 中的属性也要可写，并且类型一致（4.1.1；4.1.2）。如果 *O* 的定义是一个 setter ，那么已 *Receiver* （实现）为 `this` 调用 setter （7）。

对 superPreperty ，*O* 最初为 base value ，*Receiver* 为 thisValue 。对普通属性来说，*O* 与 *Receiver* 本来时一致的。但是，经过 2.2.1 递归进入原型链之后，*O* 将为原型对象，但 *Receiver* 依然为原调用对象。

### 函数调用

当 Reference *V* 的值是一个函数，直接调用这个函数时，函数中的 `this` 绑定为 GetThisValue(*V*) （参见 [EvaluateCall](https://www.ecma-international.org/ecma-262/#sec-evaluatecall)）。对 superProperty 来说，getThisValue(*V*) 的结果就是 SuperProperty 所在环境的 `this` 。

### base value 与 thisValue

对于 superProperty ，base value 提供了属性定义，thisValue 为实现。通常情况下（不去故意改变 `this` 绑定时），base value 会处于 thisValue 的原型链中。

对于 `class Derived extends Base` 的非 `static` 成员方法，`this` 通常为 `new Derived` 得到的对象，[[HomeObject]] 为 `Derived.prototype` ，base value 为 `Object.getPrototypeOf(`[[HomeObject]]`) === Object.getPrototypeOf(Dereived.prototype) === Base.prototype`。而 `Object.getPrototypeOf(this) === Dreived.prototype`。即 `Object.getPrototypeOf(Object.getPrototypeOf(`thisValue`)) ===` base value ；base value 是 thisValue 的原型对象的原型对象。

对 `static` 成员方法，`this` 通常为 `Derived` ；[[HomeObject]] 也时 `Derived` 。base value 时 [[HomeObject]] 的原型对象（`Base`），也就是 thisValue 的原型对象。

对 object literal 内直接定义的方式，`this` 于 [[HomeObject]] 通常也是一致的，为 object 本身。从而，base value 为 thisValue 的原型对象。

`this` 绑定是在函数调时决定的。但是 base value （[[HomeObject]]）时在函数定义时就已经确定的。所以，如果在函数调用时通过各种途径，改变了 `this` 通常的取值，那么上述关系将不再成立。

### 小结

javascipt 中对对象属性的读写，都要通过原型链先找到属性的定义，然后根据定义在实际对象中读写。对普通对象属性来说，查找的起点与被读写的对象是同一个对象。

对于 superProperty ，定义的查找是从基类的原型对象（类成员非 `static` 方法），或者 `this` 的原型对象（object literal，类 `static` 成员方法）开始的。而实际读写的对象都是 `this` 。由于查找是从原型对象开始的，所以“派生类”的属性是找不到的。派生类的实例中的属性也是找不到的。

可以认为，superProperty 只是改变的属性定义查找的起点，从而只有基类中的属性才可以被找到，但实际操作的对象依然为 `this` 。

1. 取值的时候，只有定义在基类/原型对象中的属性可以被读取到。定义在派生类/类实例中的属性是不可见的。（会返回 `undefined`）
   * 如果定义为 getter ，会以当前的 `this` 来调用 getter 。
2. 赋值的时候，值实际会被写入 `this` 。
   * 从而，对 superProperty 赋值之后再读取，其结果依然可能是 `undefined` 。
   * 如果定义为 setter ，会以当前的 `this` 来调用 setter 。
3. 直接函数调用，函数中的 `this` 绑定，为 superProperty 所处环境的 `this`。
