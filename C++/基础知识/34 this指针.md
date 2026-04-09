---
data: 2025-12-17T14:42:00
tags:
---
1. this 指针
	- 本质：成员函数隐藏形参 `类名 *const this`，由编译器自动传入，指向调用该函数的对象地址；函数体内凡写成员名即被转写成 `this->成员`。
	- 用途：
		- 区分同名形参与数据成员：`void set(int x){ this->x = x; }`
		- 链式调用：返回 `*this` 可连续 `.` 调用，如 `obj.setA(1).setB(2);`
		- 自比较/自返回：`const Stock& topVal(const Stock& a) const` 中，用 `*this` 代表当前对象，与参数 a 比较后返回价值更大者
	- 常量性：在 const 成员函数里，this 被隐式声明为 `const 类名 *const this`，故无法通过它修改对象内容（mutable 成员除外）
	- 生命周期：this 仅在非静态成员函数执行期间有效，不能保存其值供后续使用，尤其避免返回局部对象的 this 指针。
2. const 的三处用法
	- `const Stock& topVal(const Stock& a) const`
	- 返回类型前的 `const`
		- `const Stock&` = 返回 “对 Stock 的 const 引用”
		- 效果：调用者只能读返回的对象，不能写。写成引用避免拷贝，加 const 禁止写
	- 参数表里的 `const`
		- `const Stock& a` = 形参 a 是 “对 Stock 的 const 引用”
		- 效果：函数体内不能修改 a，同时引用传参避免拷贝开销
	- 参数表后的 `const`（只能对成员函数用）
		- `... ) const` = 把隐式形参 `this` 声明成 `const Stock* const this`
		- 效果：函数体里不能修改当前对象的任何数据成员（mutable 成员除外），因而可被 const 对象调用
	- 总结：“前”管返回，“中”管形参，“尾”管自己；前中 const 都是“只读引用”，尾 const 是“对象只读承诺”
