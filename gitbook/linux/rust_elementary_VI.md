# Rust 基础 VI

多线程, 面向对象, 匹配模式, unsafe, 高级 trait, 高级类型, 高级函数与闭包, 宏



## 多线程

concurrent: 不同代码独立运行

parallel: 不同代码同时运行

process: 代码运行在进程中

thread: 代码每个独立部分由不同线程同时运行

通过 `thread::spawn` 创建线程, 返回类型 `JoinHandle`, 可以使用其 `join` 方法阻塞当前线程并等待 handle 所指线程完成.

```rust
use std::thread;
use std::time::Duration;

fn main() {
    let handle = thread::spawn(|| {
        for i in 1..10 {
            println!("{} from spawned thread!", i);
            thread::sleep(Duration::from_millis(1));
        }
    });
    
    for i in 11..15 {
        println!("{} from main thread!", i);
        thread::sleep(Duration::from_millis(1));
    }
    
    handle.join().unwrap(); // block main thread until spawned threads done
}
```

使用 `move` 转移线程间值的所有权

```rust
use std::thread;

fn main() {
    let v = vec![1, 2, 3];
    // move v ownership to spawned thread 
    let handle = thread::spawn(move || {
       println!("vector: {:?}", v); 
    });
    
    handle.join().unwrap();
}
```

另一种传递数据的方法, 使用**消息传递**. 通常使用 `mpsc::channel` 创建用于消息传递的管道 (mpsc: multiple producer, single consumer). 其返回类型为 `(sender, receiver)`.

```rust 
use std::sync::mpsc;
use std::thread;

fn main() {
    let (sx, rx) = mpsc::channel();
    
    thread::spawn(move || {
        let val = String::from("sending message from sender");
        sx.send(val).unwrap();
    });
    
    let received = rx.recv().unwrap();
    println!("message received: {}", received);
}
```

`send` 参数为发送的数据, 返回类型为 `Result<T, E>` 如果发送端关闭则返回错误. `recv` 会阻塞当前线程, 直到 channel 传递一个值, 返回类型为 `Result<T, E>`, 如果接收端关闭则返回错误. `try_recv` 不会阻塞线程, 有数据到达返回 `Result<T, E>`, 否则返回错误, 通常使用循环逻辑调用 `try_recv`. 注意: 发送的数据所有权会移交到接收端线程, 所以当前线程无法再使用该数据. 



rust 使用 mutex (mutual exclusion) 控制共享状态的并发, 同一时刻 mutex 只允许一个线程访问目标数据. 线程要访问数据, 需要先获取 mutex. 创建互斥锁使用 `Mutex::new(data)`, 返回类型为智能指针 `Mutex<T>` 提供内部可变性. 访问数据前, 使用 `lock` 方法阻塞线程, 获得锁, 返回类型为智能指针 `MutexGuard` (已实现 Deref 和 Drop traits). 线程之间使用 `Arc` (async rc) 来实现互斥锁, `Arc` 实现原子操作的互斥锁, 但相比于 `Rc` 性能有损失. 

```rust
use std::sync::{Mutex, Arc};
use std::thread;

fn main() {
    let counter = Arc::new(Mutex::new(0));
    let mut handles = vec![];
    
    for _ in 0..10 {
        let counter = Arc::clone(&counter);
        let handle = thread::spawn(move || {
            let mut num = counter.lock().unwrap();
            *num += 1;
        });
        handles.push(handle);
    }
    
    for handle in handles {
        handle.join().unwrap();
    }
    
    println!("Result: {}", *counter.lock().unwrap());
}
```

`RefCell<T>` 改变 `Rc<T>` 的数据, 而 `Mutex<T>` 改变 `Arc<T>` 的数据. 注意, `Mutex<T>` 有死锁风险. 



rust 语言中有两个并发实现 `std::marker::Sync` 和 `std::marker::Send` 两个 trait. 在线程间传递所有权, 需要方法实现 `Send` trait, 而 rust 里几乎所有类型 (除原始指针) 已实现 `Send`, 除了只用于单线程的 `Rc<T>`. `Sync` 安全地被多个线程引用, 假设类型 T 为 `Sync`, `&T` 则为 `Send`. 所有基础类型都是 `Sync`, 所有由 `Sync` 类型组成的类型也为 `Sync`. `Rc<T>`, `RefCell<T>`, `Cell<T>` 及其家族不是 `Sync`. 





## 面向对象

rust 不完全是面向对象的语言. 面向对象的定义: 1.面向对象的程序由对象组成; 2.对象包装数据及其方法或操作.

rust 中 `struct` 和 `enum` 类型包含数据, 并且 `impl` 提供方法, 但 `struct` 和 `enum` 并非称为对象.

封装: 调用对象的外部代码不能直接访问对象内部实现细节, 只能通过对象公开的 API 实现交互. 在 rust 中使用 `pub` 关键字声明公开方法.

继承: 对象可以沿用另外一个对象的数据和方法, 无需重复定义相关代码. rust 中则没有继承, 代码复用是通过默认 `trait` 方法共享代码. 与使用对象实现的差别在于, rust 中不能为 trait 对象添加数据, trait 用于抽象共有行为, 通用性不如传统面向对象语言中的对象.

多态: 子类型可以和父类型同时应用. rust 中使用泛型和 trait 约束 (或者称为限定参数化多态 bounded parametric) 实现多态. 将 trait 约束作用于泛型时, rust 编译器会执行单态化并静态派发 (static dispatch) 在编译阶段就确定泛型实际的具体类型. 如果编译阶段无法实现单态化, rust 编译器会进行动态派发 (dynamic dispatch) 在编译时产生额外代码, 覆盖运行所需的各种可能类型. 动态派发会导致运行时开销, 以及编译器无法内联方法代码, 妨碍部分优化操作的进行. 

rust 采用规则判定对象安全: 1.方法返回类型不为 `self`; 2.方法不包含泛型参数.

```rust
pub trait Draw {
    fn draw(&self);
}

// unsafe object trait
pub trait Clone {
    fn clone(&self) -> Self;
}

pub struct Screen {
    // Vec elements type should contain Clone trait
    pub components: Vec<Box<dyn Clone>>,
}
```

rust 实现面向对象的设计模式例子 (...) : 状态模式

值的内部状态由数个状态对象表达, 值的行为由内部状态决定. 业务需求改变时, 不需修改持有状态的值的代码, 只需要更新状态对象内部的代码, 改变其规则, 或者增加新的状态对象. 



总结: rust 支持面向对象的设计模式, 但面向对象的设计模式未必是 rust 的最佳实践, 因为 rust 具有其他语言没有的**所有权**特性.





## 匹配模式

匹配是 rust 中的特殊语法, 用于匹配简单和复杂类型的结构. 模式 (pattern) 包括: 字面值, 数组, `enum`, `struct`, `tuple`, 变量, 通配符, 占位符. `_` 用于匹配任何值, 不会绑定变量, 通常用于 match 的 default 情况.

```rust
match value {
    pattern1 => expression1,
    pattern2 => expression2,
    ...
    _ => default,
}
```

条件表达式 `if let` 等价于只有一个匹配项的 `match`. 

```rust
fn main() {
    let fcolor: Option<&str> = None;
    let flag = false;
    let age: Result<u8, _> = "233".parse();
    
    if let Some(color) = fcolor {
        println!("use favorite color {} as background", color);
    } else if flag {
        println!("flag is true, use purple as background color");
    } else if let Ok(age) = age {
        println!("use red as background color");
    } else {
        println!("use blue as background color");
    }
}
```

`while let` 条件循环表达式

```rust
fn main() {
    let mut stack = Vec::new();
    
    stack.push(1);
    stack.push(2);
    stack.push(3);
    
    while let Some(top) = stack.pop() {
        println!("stack value: {}", top);
    }
}
```

`for` 循环之后的表达式也是模式, 比如 `for (index, value) in v.iter().enumerate() {...}` 中的 tuple `(index, value)`. 

单 `let` 语句之后跟随的也是模式, `let pattern = expression`. 

```rust
let (x, y, z) = (2, 3, 3); // correct pattern
let (x, y) = (2, 3, 3); // wrong pattern
```

同理, 函数参数也可以是模式, 如 `print_position(&(x, y): &(i32, i32)) {...}` 模式为 `&(x, y)`.



可辩驳性 (Rebuttability | (ir)refutable pattern): 模式匹配失败的处理. 能匹配完备情况的模式, 为无可辩驳的, 如 `let x = 1`; 不能完备匹配所有值情况的模式, 为可辩驳的, 如 `if let Some(x) = value`. 函数参数, `let`, `for` 循环只能接受无可辩驳的模式. `if let` 和 `while let` 可接受可辩驳的模式. 



多重模式, `|` 来或多个模式, `..=` 用来表示值的范围

```rust
fn main() {
    // let x: Box<dyn std::any::Any> = Box::new(233);
    let x: Box<dyn std::any::Any> = Box::new('Z');

    if let Some(v) = x.downcast_ref::<i32>() {
        match v {
            0 | 1 => println!("The value is 0 or 1"),
            2..=9 => println!("The value is between 2 and 9"),
            _ => println!("not 0 to 9"),
        }
    }
    
    if let Some(v) = x.downcast_ref::<char>() {
        match *v {
            'A'..='Z' => println!("The value is an uppercase letter"),
            'a'..='z' => println!("The value is a lowercase letter"),
            _ => println!("any case"),
        }
    }
}
```

匹配嵌套的类型

```rust
enum Color {
    Rgb(i32, i32, i32),
    Hsv(i32, i32, i32),
}

enum Message {
    Quit, 
    Move {x: i32, y: i32},
    Write(String),
    ChangeColor(Color),
}

fn main() {
    let msg = Message::ChangeColor(Color::Hsv(0, 160, 255));
    
    match msg {
        Message::ChangeColor(Color::Rgb(r, g, b)) => {...}
        Message::ChangeColor(Color::Hsv(h, s, v)) => {...}
        _ => (),
    }
}
```

忽略值的语法, `_`, `_` 配合其他模式, `_` 开头的名, `..` 忽略值的剩余部分

```rust
struct Point {
    x: i32, 
    y: i32, 
    z: i32,
}

fn main() {
    let numbers = (1, 2, 3, 4, 5);
    // ignore second and fouth
    match numbers {
        (first, _, third, _, fifth) => {
        	println!("first is {}, third is {}, fifth is {}", first, third, fifth);
        }
    }
    
    let origin = Point{x: 0, y: 0, z: 0};
    match origin {
        Point {x, ..} => println!("x is {}", x), // ignore remains y and z
    }
    
    let numbers = (2, 4, 6, 8, 10);
    match numbers {
        // ignore middle numbers
        (first, .., last) => {
            println!("first is {}, last is {}", first, last);
        }
    }
}
```

match 守卫, pattern 额外跟随的 `if` 条件

```rust
fn main() {
    let num = Some(4);
    
    match num {
        Some(x) if x < 5 => println!("less than 5: {}", x),
        Some(x) => println!("greater than or equal to 5: {}", x),
        _ => (),
    }
}
```

`@` 符号保存匹配模式的值

```rust
struct Message {
    id: i32,
    content: String,
}

fn main() {
    let msg = Message{id: 3, content: String::from("anything")};
    
    match msg {
        Message{
            id: id_saved @ 0..=3, ..
        } => println!("Found an id in range 0 to 3: {}", id_saved),
        _ => println!("Found an id out range 0 to 3"),
    }
}
```





## unsafe

unsafe rust 存在的原因: rust 编译器静态分析是保守的, 可能会误判安全代码为错误的; 计算机硬件不安全, 那么需要 unsafe 来进行底层系统编程. `unsafe` 关键字声明的块中可进行的操作:

+ 解引用原始指针 (unsafe 中引用检查依然会执行)
+ 调用 unsafe 函数或方法
+ 访问或修改可变静态变量
+ 实现 unsafe trait

```rust
unsafe fn danger() {
    println!("do something dangerous");
}

fn main() {
    let mut num = 6;
    let r1 = &num as *const i32;
    let r2 = &mut num as *mut i32;
    
    unsafe {
        println!("r1: {}", *r1);
        *r2 = 666;
        println!("r2: {}", *r2);
    }
    
    unsafe {
        danger();
    }
    
    let address = 0x123456usize;
    let r = address as *const i32;
    
    unsafe {
        println!("r: {}", *r);
    }
}
```



`extern` 关键字用来声明外部函数接口 (FFI, Foreign Function Interface), 类似的有应用二进制接口 (ABI, Application Binary Interface). 在 rust 中实现提供其他编程语言调用的接口, 使用 `#[no_mangle]` 防止编译器改变函数名导致可读性下降. 

```rust
#[no_mangle]
pub extern "C" fn call_rust_function() {
    println!("function implemented by rust");
    // do something
}
```





## 高级特性

关联类型 `associated type` 是 trait 中的类型占位符, 用于 trait 方法的签名. 无需知道具体类型, 但包含这种不确定类型的 trait 就会用到关联类型的特性. 不同于泛型, 每次实现 trait 时, 泛型需要确定具体的类型, 但是关联类型可以一直维持类型不确定.

```rust
pub trait Iterator {
    type Item; // associated type
    
    fn next(&mut self) -> Option<Self::Item>;
}
```

默认类型参数以及 `std::ops` 运算符重载. rust 中不允许重载运算符, 但是可以通过改变运算符的默认类型, 并为特定类型实现 trait 来变相实现运算符重载.

```rust
use std::ops::Add; // default Add type is elementary type

#[derive(Debug, PartialEq)]
struct Point {
    x: i32,
    y: i32,
}

impl Add for Point {
    type Output = Point;
    
    fn add(self, other: Point) -> Point {
        Point {
            x: self.x + other.x,
            y: self.y + other.y,
        }
    }
}
```



`type` 关键字定义类型别名

```rust
type Kilometers = i32;

fn main() {
    let x: i32 = 1;
    let y: Kilometers = 2;
    println!("x + y = {}", x + y);
}
```



`!` 称为 never type, 用来充当不返回函数的返回类型, 可以理解为空类型. 

```rust
fn bar() -> ! {...} // return type is !

print!("..."); // return type is !
println!("..."); // return type is !
```



函数指针, 用于将函数传递给其他函数. 传递函数时, 函数类型会强制转换为 `fn` 即 function pointer 类型. 

```rust
fn add_x_y(x: i32, y: i32) -> i32 {
    x + y
}

fn add_twice(func: fn(i32, i32) -> i32, argx: i32, argy: i32) -> i32 {
    func(argx, argy) + func(argx, argy)
}

fn main() {
    let ans = add_twice(add_x_y, 1, 2); // ans should be (1 + 2) + (1 + 2) == 6
    println!("the answer is {}", ans);
}
```

与闭包的联系, 都需要实现函数要求的相关 traits, 才可以作为参数传入其他函数

```rust
fn main() {
    // closure
    let list_numbers = vec![1, 2, 3];
    let list_strings: Vec<String> = list_numbers.iter().map(|| i.to_string()).collect();
    
    // function pointer
    let list_numbers = vec![1, 2, 3];
    let list_strings: Vec<String> = list_numbers.iter().map(ToString::to_string).collect();
}
```

可以看 `map` 的定义, 需要传入的函数指针实现了 `FnMut` trait.

```rust
fn map<B, F>(self, f: F) -> Map(Self, F)
where
	Self: sized,
	F: FnMut(Self::Item) -> B,
{...}
```



返回闭包, 需要返回实现 trait 的具体类型 (确定大小), 一般用 Box 作为实现闭包逻辑的具体类型. 

```rust
fn ret_closure() -> Fn(i32) -> i32 {...} // error

fn ret_closure() -> Box<dyn Fn(i32) -> i32> {
    Box::new(|x| x + 1) // correct
}
```



宏 macro

本质上宏属于元编程 (metaprograming), 编写生成其他代码的代码. 与函数的区别, 函数在声明时确定参数个数和类型, 但是宏可以处理可变参数. 函数可以任意位置定义和使用, 宏需要再使用前定义在当前文件或者从其他文件引入当前作用域. 过程宏的定义, 需要提供相关宏的包, 例如 `#[proc_macro]`, 并跟随生成代码的函数. 

过程宏类型

+ 自定义派生
+ 属性宏
+ 函数宏



```rust
// function macro example
let sql = sql!(SELECT * FROM posts WHERE id=1);

#[proc_macro]
pub fn sql(input: TokenStream) -> TokenStream {...}
```

