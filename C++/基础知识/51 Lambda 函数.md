---
data: 2025-12-21T13:40:00
tags:
---
1. Lambda 函数（C++11 闭包）
	- 本质
		- 编译器自动生成**未命名函数对象**（闭包类型），可捕获局部变量、可拷贝、可存储
	- 语法四部分
		- `[捕获](参数) -> 返回类型 { 函数体 }`
		- 返回类型可省：编译器从`return 表达式`推导（若有多条`return`必须同类型）
	- 捕获列表速查
		- | 形式 | 含义 |
		  |:---:|---|
		  | `[]` | 不捕获任何局部变量 |
		  | `[x]` | **按值**捕获变量 `x`（只读默认） |
		  | `[&x]` | **按引用**捕获 `x`，可改原值 |
		  | `[=]` | **按值**捕获**全部**局部变量 |
		  | `[&]` | **按引用**捕获**全部**局部变量 |
		  | `[x, &y]` | 混合捕获 |
		  | `[this]` | 捕获当前类实例指针 |
		  | `[=, &x]` | 默认按值，但 `x` 按引用（C++14 起可更细） |
	- 代码示例
		- ```cpp
			  const long Size = 390'000;
			  std::vector<int> numbers(Size);
			  std::generate(numbers.begin(), numbers.end(), std::rand);
			  // ① 计数被 3 整除（按值捕获，无状态）
			  int count3 = std::count_if(numbers.begin(), numbers.end(),
		                                 [](int x){ return x % 3 == 0; });
			  // ② 计数被 13 整除（lambda 内改外部变量，必须引用捕获）
			  // & 捕获 count13
			  int count13 = 0;
			  std::for_each(numbers.begin(), numbers.end(),
							  [&](int x){ count13 += (x % 13 == 0);});
		  ```
	- 可调用对象存储
		- `auto f = [](int x, int y){ return x + y; };`
		- `std::cout << f(3, 6);          // 9`
		- `std::function<int(int,int)> g = f;  // 可拷贝/存储`
	- 与函数指针/对象对比
		- | 特性 | 普通函数 | 函数对象 | Lambda |
		  |---|---|---|---|
		  | 内联优化 | 依赖编译器，较难 | 依赖编译器，易内联 | 同函数对象     |
		  | 状态捕获 | 需借助全局/static | 成员变量 | 捕获列表 |
		  | 定义位置 | 文件作用域 | 类体外 | 函数体内 |
		  | 可读性 | 高 | 中 | 极高（就地定义）|
	- 性能
		- 与普通函数对象零成本抽象相同；编译器把 lambda 体直接内联到调用点
	- 常见陷阱
		- 按引用捕获离开作用域后悬空 → `UB`
		- 按值捕获默认 `const`，函数体不能改，需要改加 `mutable`
		- `int c = 0; auto cnt = [c]() mutable { return ++c; };//每次调用c自增`
	- C++14/17 后续扩展（提及）
		- 泛型 lambda：`[](auto x, auto y){ return x + y; }`
		- 初始化捕获：`[v = std::move(vec)]() { /* 使用 v */ }`
		- `constexpr lambda`：编译期可调用
		- 捕获结构化绑定：`[x = tup.first](){ ... }`
	- 总结
		- lambda = 就地定义的轻量函数对象，捕获列表控制变量生命周期与权限