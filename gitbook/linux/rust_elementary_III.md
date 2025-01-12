# Rust 基础 III

枚举, match/if let, Package/Crate/Module, Vector/String/HashMap



## 枚举

定义枚举类型, 其中的类型可以有差别, 并可以附加值. 同 `struct`, `enum` 类型也可以定义方法.

```rust
enum IpAddrKind {
    V4(u8, u8, u8, u8), 
    V6(String),
}

enum Message {
    Quit,
    Move {x: i32, y: i32},
    Write(String),
    ChangeColor(i32, i32, i32),
}

impl Message {
    fn call(&self) {
        println!("test call");
    }
}


fn main() {
    let ip4 = IpAddrKind::V4(127, 0, 0, 1);
    let ip6 = IpAddrKind::V6(String::from("::1"));
    
    route(ip4);
    route(ip6);
    
    let q = Message::Quit;
    let m = Message::Move{x:12, y:13};
    let w = Message::Write(String::from("fa1c4"));
    let c = Message::ChangeColor(0, 255, 128);
    
    m.call();
}

fn route(ip_kind: IpAddrKind) {
    println!("test route");
}
```



Rust 没有 Null, 使用枚举类型 Option\<T\> 来定义 Null, 其包含在预导入模块 (Prelude) 中, 可以直接使用 

```rust
enum Option<T> {
    Some(T),
    None,
}
```



`match` 关键字可以匹配模式对应的代码, 包括字面值, 变量名, 通配符等. match 匹配时必须**穷举所有可能**, 可以用 `_` 通配符放在最后表示 "default". 

```rust
enum Coin {
    Penny,
    Nickel,
    Dime,
    Quarter,
}

fn value_of_cents(coin: Coin) -> u8 {
    match coin {
        Coin::Penny => {
            println!("Penny");
            1
        }
        Coin::Nickel => {
            println!("Nickel");
            5
        }
        Coin::Dime => 10,
        Coin::Quarter => 25,
    }
}
```

`if let` 可以用来处理特殊枚举值的情况

```rust
fn main() {
    let v = Some(0u8);
    match v {
        Some(3) => println!("three"),
        _ => println!("others"),
    }
    
    // equal to 
    if let Some(3) = v {
        println!("three");
    } else {
        println!("others");
    }
}
```



## Package & Crate & Module

Rust 的模块系统:

+ Package (包): Cargo最顶层组织为Package | 包含1个 `Cargo.toml` 描述如何构建 Crates | 最多包含1个 library crate | 包含任意数量 binary crate | crate 数量 >= 1
+ Crate (单元包): 模块树, 产生 library 或者 binary | Crate Root 为源代码文件, Rust 从 Crate Root 开始构建 Module
+ Module (模块): 管理代码的作用域, 私有属性等 | 在 Crate 内对代码分组, 控制 item 的私有性 (public 或者 private) | 使用关键字 `mod` 构建 | 可嵌套定义
+ Path (路径): 为 struct, function 和 module 等命名的方式



rust 中索引条目需要用**路径**

+ 相对路径: 从 creat root 开始, 使用 crate 名或字面值
+ 绝对路径: 从当前 module 开始, 使用 `self`, `super` 或当前 module 的标识符
+ 标识符之间使用 `::` 分隔



私有边界 (privacy boundary)

+ 模块以内默认私有
+ Rust 所有条目默认为私有, 如 function, method, struct, enum, module, constant 等. 使用关键字 `pub` 设为公有. `pub struct` 只有名为公有, 字段 (field) 也需要显示声明为公有, 否则默认为私有. 与 struct 不同的, `pub enum` 所有枚举都是公共的.
+ 祖 module 不能访问子 module 的私有条目
+ 子 module 可以访问所有祖 module 的条目 | 使用 `super` 关键字访问

```rust
// in crate root lib.rs
fn outside_func() {}

mod private_mod {
    fn inside_func() {}
    
    fn call_outside_func() {
		super::outside_func();
        // or use absolute path
        crate::outside_func();
        
        // directly calling private func
        inside_func();
    }
}
```

需要使用私有条目时, 使用 `use` 关键字引入其祖 module 或者为 struct, enum 则引用本身到当前作用域. 如果出现重名的情况, 引用其不重名的祖 module.

使用 `pub use` 重新导出私有条目到外层作用域. 

使用嵌套路径, 简化 use 的条目 `use path_same::{path_diffs};`, 比如 `use std::io::{self, Write}`, `use std::collections::*`.



module 的多文件(夹)组织

+ Rust 会从 module 同名的文件中加载内容
+ 如果多层级 module 定义, 则从 module 同名的文件夹目录中加载子 module 及其内容 

```rust
// in lib.rs (crate root)
mod parent_module;

// in parent_module.rs
mod child_module;

// in parent_module directory, in child_module.rs
fn child_function() {}; 
// ...
```



## Vector

使用 `Vec<T>` 声明 vector 变量, 注意只能存储相同类型的值, 并在内存中连续存放. 

对于 vector 的索引, 有引用 `&v[xx]` 和 `.get(xx)` 两种方式, 前者对越界的处理是 panic, 后者会返回 `Option<T>` 类型由程序员处理. 同一作用域内, 一个 Vec 类型不能同时借用为可变和不可变类型, 只能为其一. 

```rust
fn main() {
    let v = vec![1, 2, 3, 4, 5];
    let index_num: &i32 = &v[10];
    println!("the indexed element is {}", index_num);
    
    match v.get(10) { // .get return option<i3> 
        Some(index_num) => println!("the indexed element is {}", index_num),
        None => println!("no such element"),
    }
}
```

遍历 Vec, 使用 `for` 循环来实现

```rust
fn main() {
	let mut v = vec![100, 33, 66];
    for i in &mut v { // i must be mut variable reference
        *i += 1;
    	println!("{}", *i);
    }
}
```



如果要在 Vec 里存储多种已知类型数据, 结合 `enum` 来实现

```rust
enum SheetCell {
    Int(i32),
    Float(f64),
    Text(String),
}

fn main() {
    let row = vec![
        SheetCell::Int(233),
        SheetCell::Text(String::from("purple")),
        SheetCell::Float(2.33),
    ];
}
```



## String

Rust 的核心只支持一种字符串类型, 字符串切片: `str` 或者 `&str`. `String` 类型来自标准库, 采用 `UTF-8` 编码, 可修改和可拥有.

```rust
let mut s = String::from("foo");
s.push_str(" bar");
let t1 = String::from(" inmutable");
let mut t2 = String::from(" mutable");
s.push_str(&t1);
s.push_str(&t2);
s.push(' ');
println!("{}", s);

let s = t1 + &t2; // fn add(self, s: &str) -> String {...}
// t1 ownership transfered into the add function and be disable at this scope
println!("{}", s);

let t1 = String::from("first");
let t2 = String::from("second");
let t3 = String::from("third");
let s = format!("{}-{}-{}", t1, t2, t3); // format! refer all parameters without getting ownership
println!("{}", s); // t1, t2, t3 are all available at the scope
```

String 类型**不支持**索引访问, 但是可以切片.



## HashMap

`HashMap<K,V>` 以键值对存储数据, `HashMap::new()` 声明, 使用 `insert` 增加数据. `collect` 可构建 `HashMap<K, V>`. 有 `copy trait` 的类型会复制, 没有则移动到 `HashMap`, 如果构建 `HashMap` 是传递引用类型, 则不会移动. 

```rust 
use std::collections::HashMap;

fn main() {
    let mut scores = HashMap::new();
    scores.insert(String::from("Red"), 1);
    scores.insert(String::from("Purple"), 2);
    
    let key_name = String::from("Red");
    let score = scores.get(&key_name);
    
    match score {
        Some(s) => println!("{}", s),
        None => println!("key not exists"),
    }
    
    // traverse the HashMap
    for (k, v) in &scores {
        println!("{}: {}", k, v);
    }
    
    // use collect() to create HashMap
    let teams = vec![String::from("Blue"), String::from("Yellow")];
    let initial_scores = vec![2, 3];
    // use '_' as default type leave rust to infer correct type while compiling
    let scores: HashMap<_, _> = teams.iter().zip(initial_scores.iter()).collect();
    for (k, v) in &scores {
        println!("{}: {}", k, v);
    }
}
```

`HashMap` 的 `entry` 获得键的 `enum` 枚举类型, 存在则返回值, 不存在则为 `Vacant` (空). 使用 `or_insert` 按情况更新 `HashMap`

```rust
use std::collections::HashMap;

fn main() {
    let mut scores = HashMap::new();
    scores.insert(String::from("Blue"), 2);
    println!("{:#?}", scores);
    scores.entry(String::from("Yellow")).or_insert(3);
    println!("{:#?}", scores);
    scores.entry(String::from("Blue")).or_insert(4); // Blue exists won't insert
    println!("{:#?}", scores);
    let tmp = scores.entry(String::from("Blue")).or_insert(0);
    *tmp += 2;
    println!("{:#?}", scores);
    
    // implement counter demo
    let text = "a b c d e f a b c";
    let mut map = HashMap::new();
    for word in text.split_whitespace() {
        let count = map.entry(word).or_insert(0);
        *count += 1;
    }
    println!("{:#?}", map);
}

/* output of the code above:
{
    "Blue": 2,
}
{
    "Yellow": 3,
    "Blue": 2,
}
{
    "Yellow": 3,
    "Blue": 2,
}
{
    "Yellow": 3,
    "Blue": 4,
}
{
    "e": 1,
    "f": 1,
    "c": 2,
    "d": 1,
    "b": 2,
    "a": 2,
}
*/
```



