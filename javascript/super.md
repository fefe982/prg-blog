# super

`super` 是一个 javascript 得关键字。也正是由于它是一个关键字，使得标准有机会为它得每一种用法单独定义行为，从而使它得用法与众不同。ECMA262 有专门的一节 [The super keyword](https://www.ecma-international.org/ecma-262/#sec-super-keyword) 来介绍 super 的用法。

## superCall

在派生类（`class Derived extends Base`）的构造里，可以通过 `super(...)` 的方式调用基类的构造函数。这个称作 [superCall](https://www.ecma-international.org/ecma-262/#prod-SuperCall)。在调用 superCall 之前，`this` 绑定并未初始化，不能使用 `this` 。superCall 会使用基类构造函数的“返回值”初始化当前的 `this` 。（这里的“返回值”，类似于 `new Base(...)` 的结果）

## superProperty

在派生类的构造，或者成员函数中，可以使用 `super[prop]` 或者 `super.property` 形式，称作 [superProperty](https://www.ecma-international.org/ecma-262/#prod-SuperProperty)


