# Rust 基础 V

测试驱动开发 (TDD), 闭包, 迭代器, 软件发布工作流, 智能指针, 高级 trait, Rc, 循环引用问题



## TDD

驱动测试开发的流程:

1. 编写一个会失败的测试, 运行测试确保按预期原因失败
2. 编写或修改代码, 让新测试通过
3. 重构代码, 确保测试始终通过
4. 返回步骤1循环





## 闭包 (closures)

闭包是**匿名函数**, 这个匿名函数可以保存为**变量**或者**参数**, 创建闭包后可以在其他作用域**使用**, 可以从其作用域内**捕获值**.

```rust
use std::time::Duration;
use std::thread;

fn generate_workout(intensity: u32, random_number: u32) {
    let expensive_closure = |num| {
        println!("calculating slowly ...");
        thread::sleep(Duration::from_secs(3));
        num
    };
    
    if intensity < 33 {
        println!("Today, do {} pushups!", expensive_closure(intensity));
        println!("Next, do {} situps!", expensive_closure(intensity));
    } else {
        if random_number == 0 {
        	println!("Take a break today");   
        } else {
            println!("Today, run for {} miles", expensive_closure(intensity));
        }
    }
}

fn main() {
    let intensity = 34;
    let random_number = 1;
    generate_workout(intensity, random_number);
}
```

闭包的定义只会为其参数和返回值确定唯一类型, 即多处使用闭包时, 传入和返回的类型都是相同的, 比如不能一处调用闭包使用 int 型, 另一处使用 String 型.

闭包包含以下 traits 的若干:

+ Fn: 不可变借用变量 (实现该 trait 也实现了 FnMut 和 FnOnce)
+ FnMut: 可变借用变量 (实现该 trait 也实现了 FnOnce)
+ FnOnce: 取得变量所有权 (所有闭包默认实现该 trait)

闭包可以捕获当前作用域的变量, 即不需要传入参数, 也能**调用相同作用域的变量**. 相比函数, 函数的作用域和定义函数的作用域不一致, 所以函数不能调用函数外的非 static 变量. 





## 迭代器 (iterators)

`iter()` 用于遍历数据结构中的元素

```rust
fn main() {
    let vec_test = vec![1, 2, 3];
    for val in vec_test.iter() {
        println!("{}", val);
    }
}
```

所有迭代器都实现了 `iterator trait`, 使用迭代器需要定义 `item` 用于确定 `next` 方法的返回类型, 迭代器只要求实现 `next` 方法. 下面是标准库中的实现

```rust
pub trait Iterator {
    type Item;
    
    fn next(&mut self) -> Option<Self::Item>;
    
    // methods with default implementations elided
}
```

几种迭代方法

+ iter 不可变引用的迭代
+ into_iter 创建迭代器会获得所有权
+ iter_mut 迭代可变的引用



消耗型适配器: `next()` 每次调用会消耗一个集合元素.

Iterator trait 中的某些方法转换迭代器类型, 比如 `collect` 可以把消耗型适配器收集到一个集合中提供后续调用.

```rust
let v1: Vec<i32> = vec![1, 2, 3];
let v2: Vec<_> = v1.iter().map(|x| x + 1).collect(); // "Vec<_>" let compiler itself infers the type of Vec elem 
```



### Zero-Cost Abstraction

零开销抽象, rust 中使用抽象时不引入额外开销. 由此, 迭代器效率比循环逻辑更高.

音频解码器例子, 编译器会将定长数组相关计算逻辑展开, 提高效率

```rust
let buffer: &mut [i32];
let coefficients: [i64; 12];
let qlp_shift: i16;

for i in 12..buffer.len() {
    let prediction = coefficients.iter()
    							 .zip(&buffer[i - 12..i])
    							 .map(|(&c, &s)| c * s as i64)
    							 .sum::<i64>() >> qlp_shift;
    let delta = buffer[i];
    buffer[i] = prediction as i32 + delta;
}
```





## 开发工作流

### release profile 自定义构建

cargo 主要有两个 profile 对 build 进行配置, 每个 profile 相互独立互不影响

+ dev profile
+ release profile

自定义 profile, 在 `Cargo.toml` 里添加 `[profile.your_profile]` 区域, 并自定义配置, build 时会覆盖掉默认配置

```toml
[profile.dev]
opt-level = 0
...

[profile.release]
opt-level = 3
...
```

对于配置的默认值和完整选项参考[官方文档](https://doc.rust-lang.org/cargo/reference/profiles.html).



### https://crates.io/ 发布库

通过发布包 (crates) 来共享源代码, crate 注册表在 `https://crates.io/`, 托管源代码并分发已注册的包

文档注释: 使用 `///` 三斜线表示文档注释, 可以使用 `cargo doc [--open]` 自动生成文档, 用浏览器打开可以查看自动生成的对每个函数的注释 (在 source code 里每个函数上方 `///` 添加的注释, 所以其实是 code 里人工完成注释, `cargo doc` 的功能相当于将代码中的文档注释自动化组织成 html 文档)

外层注释: `//!` 用于添加外层条目也就是 crate 或者 module 的注释, 而不是对紧随其后的条目 (function) 进行注释



crate 的程序结构在开发时合理组织, 可能对使用者不够方便, 比如 `A_crate::B_module::C_module::TargetType` 是开发者开发时组织代码的结构, 但是使用者要寻找到目标类型并调用, 则需要深入代码模块, 不够方便, 最理想的情况是直接可以从最上层的 crate 完成调用 `A_crate::TargetType`. 由此 rust 提供 `pub use` 导出方便调用的公共 `API`, 该功能重新导出一个与内部私有结构不同, 但逻辑一致的对外公共结构.

```rust
// in crate root (lib.rs)
pub use self::a_module::b_module::TargetTypeA;
pub use self::c_module::d_module::e_module::TargetTypeB;
// pub use ...;

// in other crate you can use TargetTypeA(B) by directly use crate::TargetType
use crate_name::TargetTypeA;
use crate_name::TargetTypeB;
// use ...;
```



用浏览器打开 `https://crates.io/`, 用 github 账号登录, `add token` 然后在本地 `cargo login personal_token`, 需要在 `Cargo.toml` 的 [package] 区域添加元数据

+ crate: 唯一名
+ description: 用于 crate 搜索
+ license: 使用的许可证, 参考 http://spdx.org/licenses/

然后 `cargo publish` 发布当前 crate. (不成功的按照提示进行操作解决)

注意: 一经发布的 crate **永久保存**于 `cargo.io`, 无法覆盖和删除. 

相同 crate 发布新版本: 修改 `Cargo.toml` 的 `version` 值, 语义版本参考 http://semver.org/. 然后执行 `cargo publish`.

撤回版本: `cargo yank --vers 1.0.1` 防止新项目依赖于发布的 crate, 旧项目已经依赖于该 crate 则不受影响. (已发布的 crate 代码依然保存于 `crates.io`)

取消撤回: `cargo yank --vers 1.0.1 --undo`



### workspaces 组织大工程

`Cargo.toml` 配置工作空间维护相互关联的 crates.

最上层 `Cargo.toml`, 配置所有 crates

```toml
[workspace]

members = [
	"crate1",
	"crate2",
	...,
]
```

在根目录下 `cargo new crate1 crate2 ...` 创建每个 crate, 并配置其中的 `Cargo.toml`, 在 `[dependencies]` 显式声明每个依赖项, 比如下面的 `crate1` 的 `Cargo.toml` 需要配置依赖于 `crate2`

```toml
[package]
...

[dependencies]
crate2 = {path = "../crate2"}
```

所有 crates 只有一个 `Cargo.lock` 在根目录, 这样确保所有依赖库 version 一致.



### https://crates.io/ 安装库

`cargo install xxx` 安装库, 只能安装具有二进制的 crate. 二进制 crate 由 `src/main.rs` 或者其他被指定为二进制文件的 crate 生成.

安装的库会自动添加到环境变量中, 在命令行可以直接使用二进制文件.



### 扩展 cargo

环境变量中的某个二进制程序可以通过 `cargo xxx` 子命令的形式调用, 列出所有自定义命令 `cargo --list`.





## 智能指针

通过引用计数 (reference counting) 记录所有者的数量, 并在没有所有者时自动清理数据.

+ `Box<T>`: 在 heap 内存上分配值
+ `Rc<T>`: 启用多重所有权的引用计数类型
+ `Ref<T>`: 通过 `RefCell<T>` 访问, 在运行时借用规则的类型, 不可变
+ `RefMut<T>`: 通过 `RefCell<T>` 访问, 在运行时借用规则的类型, 可变



### Box

最简单的智能指针, stack 上指向 heap 的指针, 没有性能开销. 实现 `Deref Trait` 解引用 和 `Drop Trait` 清理. 在**编译时**检查借用规则. 

使用场景:

+ 编译时无法确定大小, 但是使用时需要确定大小
+ 大量数据移交所有权, 但是不复制
+ 使用值时只关心 trait, 而不关心类型



为 Box 智能指针实现 Deref Trait 

```rust
use std::ops::Deref;

struct fBox<T>(T); // Box<T> is tuple struct
impl<T> fBox<T> {
    fn new(x: T) -> fBox<T> {
        fBox(x)
    }
}

impl<T> Deref for fBox<T> {
    type Target = T;
    
    fn deref(&self) -> &T {
        &self.0
    }
}

fn main() {
    let x = 233;
    let y = fBox::new(x);
    // let y = Box::new(x);
    // let y = &x;
    
    assert_eq!(5, x);
    assert_eq!(5, *y); // is equivelent to *(y.deref())
}
```

Deref Coercion 隐式解引用转化, 当传入函数的变量类型和函数参数类型不匹配时, 就会发生隐式解引用转化 (在编译时完成, 不会产生额外性能开销). 简单理解, rust 会在编译时反复调用 `deref()` 直到传入类型和参数类型一致.

隐式解引用规则:

+ T: Deref<Target=U>, 允许 &T 转换为 &U (都是常量)
+ T: DerefMut<Target=U>, 允许 &mut T 转换为 &mut U
+ T: Deref<Target=U>, 允许 &mut T 转换为 &U, 但不允许 &T 转换为 &U (常量不能转变量)



Drop Trait, 定义当值离开作用域时进行的操作. rust 不允许人工调用 `drop()`, 只能值离开作用域时自动调用. 不过允许调用 `std::mem::drop` 来主动 drop 值.



## Rc

引用计数的智能指针 `Rc<T>`, Reference Counting. 当引用数为0时, 会自动清理. 

使用场景:

+ 需要在 heap 上分配数据, 且无法确定不同函数对数据的使用顺序
+ 仅在单线程适用

注意, `Rc<T>` 不在预导入模块中, 需要主动导入库; `Rc::clone(&a)` 会增加计数; `Rc::strong_count(&a)` 获得引用计数. 默认为不可变引用, 只能读不能写.

```rust
use std::rc::Rc;
```

`Rc::clone()` 和 `clone()` 的区别, 前者只增加引用计数, 不会进行数据深度拷贝, 后者通常会进行深度拷贝.

`Rc<T>` 使用内部可变性, 实现不可变引用的数据修改 (unsafe)

内部可变模式 (Interior Mutability Pattern): 不可变类型提供可修改其内部值的API. 



`RefCell<T>` 是唯一所有权的引用, 只在**运行时**检查借用规则, 不满足借用规则会 panic. 用于绕过 rust 编译器的保守借用规则检查.

+ `borrow` 返回智能指针 `Ref<T>`, 已实现 `Deref`, 每次调用时, 不可变技术+1, 离开作用域-1
+ `borrow_mut` 返回智能指针 `RefMut<T>`, 已实现 `Deref`, 每次调用时, 可变技术+1, 离开作用域-1



<img src="assets/image-20250212121402854.png" alt="image-20250212121402854" style="zoom: 50%;" />





### 循环引用

引用循环 (Reference Cycles): 可能泄露内存, 需要防止内存泄露的发生. rust 也有可能出现内存泄漏, 那就是使用 `Rc<T>`, `RefCell<T>` 可以构造循环引用引发内存泄露. 出现循环引用时, 每个项的引用不会归零, 所以一直不清理回收.

```rust
fn main() {
    let a = Rc::new(Cons(5, RefCell::new(Rc::new(Nil))));
    let b = Rc::new(Cons(10, RefCell::new(Rc::clone(&a))));
    if let Some(link) = a.tail() {
        *link.borrow_mut() = Rc::clone(&b);
    }
}
```

上面代码构造了 a 和 b 之间的循环引用.

防止循环引用的方法: `Weak<T>` 弱引用不影响引用计数. 调用 `Rc::downgrade` 创建 Weak Reference 并使 weak_count + 1. 弱引用转换为强引用, 使用 `Rc::upgrade` 方法.

```rust
use std::cell::RefCell;
use std::rc::{Rc, Weak};

#[derive(Debug)]
struct Node {
    value: i32,
    parent: RefCell<Weak<Node>>,
    children: RefCell<Vec<Rc<Node>>>,
}

fn main() {
    let leaf = Rc::new(Node {
        value: 2,
        parent: RefCell::new(Weak::new()),
        children: RefCell::new(vec![]),
    });
    
    let branch = Rc::new(Node {
        value: 33,
        parent: RefCell::new(Weak::new()),
        children: RefCell::new(vec![Rc::clone(&leaf)]),
    });
    
    // use weak reference to avoid reference cycle
    *leaf.parent.borrow_mut() = Rc::downgrade(&branch);
}
```

