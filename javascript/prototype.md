# 原型对象

这里不介绍原型链。

javascript 中有若干长得跟 "prototype" / "proto" 很想的属性/函数，这里简单总结一下他们都是啥，哪个是原型对象，哪个不是。

## [[Prototype]]

这个对象的一个内置槽，对程序员是不可见（不能直接操作）的。[[Prototype]] 记录了 javascipt 对象的原型对象。

ECMA262 关于它有如下[介绍]([9.1Ordinary Object Internal Methods and Internal Slots](https://www.ecma-international.org/ecma-262/#sec-ordinary-object-internal-methods-and-internal-slots))：

> All ordinary objects have an internal slot called [[Prototype]]. The value of this internal slot is either `null` or an object and is used for implementing inheritance. Data properties of the [[Prototype]] object are inherited (and visible as properties of the child object) for the purposes of get access, but not for set access. Accessor properties are inherited for both get access and set access.

## [[GetPrototypeOf]]

这也是对象的一个内置槽，程序员无法直接操作。它是一个方法，用于获取对象的 [[Prototype]] 槽。

在 ECMA262 中，除了 Proxy 类型的对象，其它对象的 [\[\[GetPrototypeOf\]\]](https://www.ecma-international.org/ecma-262/#sec-ordinary-object-internal-methods-and-internal-slots-getprototypeof) 直接返回 [[Prototype]] 的值。

## `Object.getPrototypeOf(O)`

Object 对象的 [getPrototypeOf](https://www.ecma-international.org/ecma-262/#sec-object.getprototypeof) 方法。这个是程序员可以调用的。当 `O` 为对象时，返回 `O.`[[GetPrototypeOf]]`()` 。

这是程序员可以使用的获取 `O` 的原型对象的方法。

## `Object.prototype.__proto__`

`obj.__proto__` 也是 javascript 里常用的一种获取原型对象的方式。它实际是一个 [getter / setter](https://www.ecma-international.org/ecma-262/#sec-object.prototype.__proto__) 。

用作 getter 时，`obj.__proto__` 返回 `obj.`[[GetPrototypeOf]]`()` ，也就是可以取得 obj 的原型对象。与 `Object.getPrototypeOf(obj)` 结果是一致的。

根据 ECMA262 ，这一属性仅在浏览器中提供。不够貌似 node 也提供了这一属性。

由于它定义在 `Object.prototype` 上，所以，仅当 `Object.prototype` 在 `obj` 的原型链上时，才可以使用这一属性。当 `Object.prototype` 不在原型链上时（例如通过 `Object.create(null)` 生成的对象），无法使用这一属性。`Object.getPrototypeOf(O)` 则无此限制。

## `prototype`

这是一个看起来很像原型对象的属性，但是，它不是所在对象的原型对象。即，`obj.prototype` 不是 `obj` 的原型对象。

构造函数都有此属性。ECMA262 中关于它的[介绍](https://www.ecma-international.org/ecma-262/#sec-function-instances-prototype)如下：

> Function instances that can be used as a constructor have a `prototype` property. Whenever such a Function instance is created another ordinary object is also created and is the initial value of the function's `prototype` property. Unless otherwise specified, the value of the `prototype` property is used to initialize the [[Prototype]] internal slot of the object created when that function is invoked as a constructor.
>
> This property has the attributes { [[Writable]]: true, [[Enumerable]]: false, [[Configurable]]: false }.

通常情况下，当使用构造函数（`F`）构造一个新对象（`obj`）时，构造函数的 `prototype` 属性（`F.prototype`），将成为新对象的原型对象（`obj.`[[Prototype]]）。

`F.prototype` 不是 `F` 的原型对象。`F.prototype` 通常情况下时 `new F(...)` 的原型对象。

## `instanceof`

`obj instanceof F` 检查 `obj` 是否是 `F` 的实例。它检查的并非 `F` 是否在 `obj` 的原型链上。这一检查要求 `F` 是一个构造函数，并检查 `F.prototype` 是否在 `obj` 的原型链上。

[InstanceOfOperator(*V*, *target*)](https://www.ecma-international.org/ecma-262/#sec-instanceofoperator) ：
> The abstract operation InstanceofOperator(*V*, *target*) implements the generic algorithm for determining if ECMAScript value *V* is an instance of object *target* either by consulting *target*'s @@hasinstance method or, if absent, determining whether the value of *target*'s `prototype` property is present in *V*'s prototype chain.

通常所说 `obj` 是 `F` 类型的对象，指的也是 `F.prototype` 在 `obj` 的原型链上。

## 常见“构造”函数的 `prototype`

| 函数 | `prototype` |
|---|----|
| 普通函数 | `Object()` |
| `class C` | `Object()` |
| `class C extends null` | `Object.create(null)` |
| `class C extends B` | `Object.create(B.prototype)` |

箭头函数与成员函数不能构造，没有 `prototype` 属性。

普通函数的 `prototype` 属性是可写的，`class` 的 `prototype` 属性是不可写的。
