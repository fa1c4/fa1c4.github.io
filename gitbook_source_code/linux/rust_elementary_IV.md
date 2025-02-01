# Rust 基础 IV

错误处理, 泛型, Trait, 生命周期, 测试

## 错误处理

通常 Rust 在编译时会提示错误, 需要程序员处理. 运行时的错误, Rust 没有异常机制, 使用两种错误分类处理

### 错误分类

+ 可恢复错误: 例如文件未找到, 可再次尝试 

  + 提供 `Result<T, E>` 进行处理

+ 不可恢复错误: BUG, 例如数组越界访问等 

  + panic

  + `panic!` 宏, 依次执行: 打印错误信息, 展开 unwind, 清理调用栈 stack, 退出程序

  + unwind 会增加程序工作量, 可以用中止 abort 替代, 缩小 binary 的大小. 设置方法, 在 `Cargo.toml` 文件中增加

    ```rust
    [profile.release]
    panic = 'abort'
    ```

  + 通过 `panic!` 的函数回溯信息定位问题代码. 相关环境变量 `RUST_BACKTRACE`. 带调试信息的回溯需要启用调试符号, 编译时不带 `--release`



Result 枚举类型

```rust
enum Result<T, E> {
    Ok(T), // operation success, T is the type of data
    Err(E), // operation fail, E is the type of error
}
```

`panic!` 示例代码

```rust
use std::fs::File;
use std::io::ErrorKind;

fn main() {
    let f = match File::open("test.txt") {
        Ok(file) => file,
        Err(error) => match error.kind() {
        	ErrorKind::NotFound => match File::create("test.txt") {
                Ok(fc) => {
                    println!("created the file");
                    fc
                },
                Err(e) => panic!("error creating file: {:?}", e),
            },
            other_error => panic!("error opening file: {:?}", other_error),
        },
    };
    
    println!("{:#?}", f);
}
```

使用闭包来简化代码 (闭包详见后面的 blog)

```rust
fn main() {
    let f = File::open("test.txt").unwrap_or_else(|error| {
        if error.kind() == ErrorKind::NotFound {
        	File::create("test.txt").unwrap_or_else(|error| {
                panic!("error creating file: {:?}", error);
            })
        } else {
            panic!("error opening file: {:?}", error);
        }
    });
}
```

也可以使用 `.expect("error message")` 来打印自定义的错误信息.



### 传播错误

将错误传递给调用者处理, `?` 运算符是 Rust 内置错误处理逻辑符, `?` 会调用 `from` trait 隐式转换 `Err(e)` 到函数定义的返回错误类型. `?` 运算符只能用于返回类型为 `Result` 的函数.

```rust
let t = <operation>?;
// <=>
let t = match <operation> {
	Ok(k) => k,
    Err(e) => return Err(e),
};
```

代码示例

```rust
use std::fs::File;
use std::io;
use std::io::Read;

fn read_from_file() -> Result<String, io::Error> {
    let mut f = File::open("test.txt")?;
    let mut s = String::new();
    f.read_to_string(&mut s)?;
    Ok(s)
}
```

**链式调用**简化代码

```rust
fn read_from_file() -> Result<String, io::Error> {
    let mut s = String::new();
    File::open("test.txt")?.read_to_string(&mut s)?;
    Ok(s)
}
```



使用 `panic` 的场景

+ 编写示例: unwrap
+ 原型代码: unwrap, expect
+ 测试: unwrap, expect



## 泛型

泛型, 简单理解为代码模板, 里面的"占位符"会在编译时替换为具体类型

```rust
fn largest<T>(list: &[T]) -> T {
    ...
}
```



struct 中定义泛型, 及其方法定义

```rust
struct Point<T> {
    x: T,
    y: T,
}

// define T type method
impl<T> Point<T> {
    fn x(&self) -> &T {
        &self.x
    }
}

// define specific type method
impl Point<i32> {
    fn x_i32(&self) -> &i32 {
        &self.x
    }
}
```



Enum 中定义泛型

```rust
enum Option<T> {
    Some(T),
    None,
}

enum Result<T, E> {
    Ok(T),
    Err(E),
}
```





## Trait

`trait` 将方法签名组合实现特定逻辑, 只包含方法签名, 不包含实现

```rust
pub trait Summary {
    fn summarize(&self) -> String;
}

pub struct NewsArticle {
    pub headline: String,
    pub location: String,
    pub author: String,
    pub content: String,
}

impl Summary for NewsArticle {
    fn summarize(&self) -> String {
        format!("{} by {} ({})", self.headline, self.author, self.location)
    }
}

pub struct Tweet {
    pub username: String,
    pub content: String,
    pub reply: bool,
    pub retweet: bool,
}

impl Summary for Tweet {
    fn summarize(&self) -> String {
        format!("{}:\n{}", self.username, self.content)
    }
}


fn main() {
	let tweet = Tweet {
        username: String::from("horse ebooks"),
        content: String::from("testing content"),
        reply: false,
        retweet: false,
    };
    
    println!("one new tweet\n{}", tweet.summarize())
}
```

trait 约束只能在当前 crate 定义, **此规则**确保其他代码不会破坏当前代码. 

默认实现的方法可以调用 trait 中其他方法 (允许没有默认实现). 



trait 作为参数. impl trait 返回类型必须唯一.

```rust
use std::fmt::Display;

pub trait Summary {...}
pub struct NewsArticle {...}
impl Summary for NewsArticle {...}
pub struct Tweet {...}
impl Summary for Tweet {...}

// impl trait
pub fn notify(item: impl Summary + Display) {
    println!("Breaking news! {}", item.summarize())
}

// trait bound
pub fn notify<T: Summary + Display>(item: T) {
    println!("Breaking news! {}", item.summarize())
}
```

使用 trait bound 有条件实现方法. 为满足 trait bound 所有类型上实现 trait 是覆盖实现 (Blanket Implementations)

```rust
use std::fmt::Display;

struct Pair<T> {
    x: T,
    y: T,
}

// any type has the method
impl<T> Pair<T> {
    fn new(x: T, y: T) -> Self {
        Self {x, y}
    }
}

// only types with traits Display & PartialOrd have the method
impl <T: Display + PartialOrd> Pair<T> {
    fn cmp_display(&self) {
        if self.x >= self.y {
            println!("x is larger");
        } else {
            println!("y is larger");
        }
    }
}
```



多 trait 约束的简化

```rust
pub fn notify<T: Summary + Display, U: Clone + Debug>(a: T, b: U) -> String {
    format!("Breaking news! {}", a.summarize())
}

// equivalence
pub fn notify<T, U>(a: T, b: U) -> String
where
	T: Summary + Display,
	U: Clone + Debug,
{
    format!("Breaking news! {}", a.summarize())
}
```





## 生命周期

生命周期: 引用保持有效的作用域. 生命周期通常是隐式可推断的. 当生命周期有不同方式互相关联时, 需要手动标注生命周期.

如果参数或者返回类型包含引用类型, 需要标注生命周期. 注意: 标注生命周期不会改变实际生命周期长度, 指定泛型生命周期参数时, 可以接收带有任何生命周期的引用. 使用 `'a` (或者 `'` 加小写字母) 标注生命周期, 且在 `&` 号后面与类型关键字用空格分开.

```rust
// real lifetime is the shortest of lifetime of parameters x & y
fn longer<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() {
        x
    } else {
        y
    }
}

fn main() {
    let string1 = String::from("test1");
    let string2 = "test233";
    let result = longer(string1.as_str(), string2);
    println!("The longer string is {}", result);
}
```



生命周期省略规则: rust 团队在大量实践后总结了可预测的生命周期模式, 编入编译器代码中, 使得开发者不必显示声明生命周期. 如果不能隐式推断, 则编译时抛出错误, 需要开发者人工标注类型的生命周期.

```rust
fn first_word<'a>(s: &'a str) -> &'a str {
    let bytes = s.as_bytes();
    for (i, &item) in bytes.iter().enumerate() {
        if item == b' ' {
            return &s[0..i];
        }
    }
    &s[..]
}

// equivelence
fn first_word(s: &str) -> &str {
    let bytes = s.as_bytes();
    for (i, &item) in bytes.iter().enumerate() {
        if item == b' ' {
            return &s[0..i];
        }
    }
    &s[..]
}
```

生命周期推断规则 (所有规则都不适用时, 则编译报错)

+ 每个引用类型都有自己的生命周期
+ 只有一个输入生命周期参数, 则所有输出参数的生命周期与该参数相同
+ 多个输入参数时, 如果包含 `&self` 或者 `&mut self`, 则所有输出参数生命周期与 `self` 参数相同



`'static` 生命周期是整个程序运行时间, 所有字符串字面值默认为 `'static` 生命周期.



复合例子

```rust
use std::fmt::Display;

fn longer_with_an_announcement<'a, T> 
	(x: &'a str, y: &'a str, ann: T) -> &'a str
where
	T: Display,
{
    println!("Announcement! {}", ann);
    if x.len() > y.len() {
        x
    } else {
        y
    }
}
```





## 测试

rust 的测试, 就是测试函数, 包括三个操作: 准备数据, 运行代码, 断言结果. 在函数上加 `#[test]`, 就可以声明函数为测试函数.

使用 `cargo test` 运行所有测试函数, rust 会构建一个 `test runner` 可执行文件并执行. 测试成功提示 `ok`, 失败提示 `panic`.



`assert!` 宏, 来自标准库, 检查测试结果, 返回 `true` 测试通过, 返回 `false` 则调用 `panic!` 测试失败. 

```rust
#[test]
fn larger_can_hold_smaller() {
    let larger = Rectangle {
        length: 8,
        width: 7,
    };
    let smaller = Rectangle {
        length: 5,
        width: 1,
    };
    
    assert!(larger.can_hold(&smaller));
}
```

可以向 `assert!`, `assert_eq!`, `assert_ne!` 添加自定义信息, 在测试失败时打印出来. 对于 `assert!` 第二个参数为自定义信息, 对于`assert_eq!`, `assert_ne!` 第三个参数是自定义信息. 自定义信息参数会被传递给 `format!` 宏, 可使用占位符 `{}` 进行字符串处理.

```rust
#[test]
fn greetings_contain_name() {
    let result = greetings("falca");
    assert!(
    	result.contains("falca"),
        "greetings do not contain name, whose value is '{}'",
        result
    );
}
```



`#[should_panic]` 用于测试预期失败的代码.

测试除了 `panic` 还可以使用 `Result<T, E>` 编写测试逻辑, 此时不要标注 `#[should_panic]`.



### 控制测试运行方式

默认行为:

+ 并行运行所有测试
+ 捕获 (即打印) 除测试相关信息之外的所有输出, 测试通过看不到 `println!` 的输出, 测试失败则会看到.

```shell
cargo test --help
Execute all unit and integration tests and build examples of a local package

Usage: cargo test [OPTIONS] [TESTNAME] [-- [ARGS]...]

Arguments:
  [TESTNAME]  If specified, only run tests containing this string in their names
  [ARGS]...   Arguments for the test binary

Options:
      --no-run                   Compile, but don't run tests
      --no-fail-fast             Run all tests regardless of failure
      --future-incompat-report   Outputs a future incompatibility report at the end of the build
      --message-format <FMT>     Error format
  -q, --quiet                    Display one character per test instead of one line
  -v, --verbose...               Use verbose output (-vv very verbose/build.rs output)
      --color <WHEN>             Coloring: auto, always, never
      --config <KEY=VALUE|PATH>  Override a configuration value
  -Z <FLAG>                      Unstable (nightly-only) flags to Cargo, see 'cargo -Z help' for details
  -h, --help                     Print help

Package Selection:
  -p, --package [<SPEC>]  Package to run tests for
      --workspace         Test all packages in the workspace
      --exclude <SPEC>    Exclude packages from the test
      --all               Alias for --workspace (deprecated)

Target Selection:
      --lib               Test only this package's library
      --bins              Test all binaries
      --bin [<NAME>]      Test only the specified binary
      --examples          Test all examples
      --example [<NAME>]  Test only the specified example
      --tests             Test all test targets
      --test [<NAME>]     Test only the specified test target
      --benches           Test all bench targets
      --bench [<NAME>]    Test only the specified bench target
      --all-targets       Test all targets (does not include doctests)
      --doc               Test only this library's documentation

Feature Selection:
  -F, --features <FEATURES>  Space or comma separated list of features to activate
      --all-features         Activate all available features
      --no-default-features  Do not activate the `default` feature

Compilation Options:
  -j, --jobs <N>                Number of parallel jobs, defaults to # of CPUs.
  -r, --release                 Build artifacts in release mode, with optimizations
      --profile <PROFILE-NAME>  Build artifacts with the specified profile
      --target [<TRIPLE>]       Build for the target triple
      --target-dir <DIRECTORY>  Directory for all generated artifacts
      --unit-graph              Output build graph in JSON (unstable)
      --timings[=<FMTS>]        Timing output formats (unstable) (comma separated): html, json

Manifest Options:
      --manifest-path <PATH>  Path to Cargo.toml
      --lockfile-path <PATH>  Path to Cargo.lock (unstable)
      --ignore-rust-version   Ignore `rust-version` specification in packages
      --locked                Assert that `Cargo.lock` will remain unchanged
      --offline               Run without accessing the network
      --frozen                Equivalent to specifying both --locked and --offline

Run `cargo help test` for more detailed information.
Run `cargo test -- --help` for test binary options.
```

如果要显示编译的相关帮助信息使用命令 `cargo test -- --help` ...



测试的规定, 由于测试并行运行, 所以要求

+ 测试之间不互相依赖
+ 不共享相同的状态 (环境, 工作目录, 环境变量等)

使用 `--test-threads` 控制线程数量, 如果要使用单线程进行测试, 设置 `--test-threads=1` 即可.



若要查看测试通过时的标准输出流, 使用 `--show-output` 来打印输出信息.



`cargo test xxx` 运行特定测试, `xxx` 为测试函数名.

运行多个测试, 使用相同前缀进行指定多个测试函数, 比如前缀为 `add` 则测试时会运行所有测试函数名前缀为 `add` 的测试.



`#[ignore]` 设置为默认忽略的测试, 使用 `cargo test -- --ignored` 单独运行默认忽略的测试.



### 测试的分类

+ 单元测试: 针对单个模块进行测试, 包括 private 接口, `#[cfg(test)]` 标注
+ 集成测试: 库外部进行测试多个模块, 只能测试 public 接口, 无标注, 追求覆盖率



#### 集成测试

创建测试目录 `tests`, 该目录下每个测试文件都是一个包 (crate). 测试文件中需要导入目标库. 

```rust
use adder;

#[test]
fn it_adds_two() {
    assert_eq!(4, adder::add_two(2));
}
```

运行特定函数 `cargo test function_name`, 运行特定测试文件所有测试 `cargo test --test file_name`.

测试文件组织规范: 测试代码复用时, 在 `tests/xxx` 目录下创建 `.rs` 文件包含需要复用的代码, 这样 `cargo test` 不会运行子模块的代码, 因为 `cargo test` 只以包为单位运行代码. 比如创建 `tests/common` 目录, 添加 `mod.rs` 文件, 包含需要复用的测试代码.

```rust
mod common;

#[test]
fn testing() {
    common::test_xxx();
}
```



只有 library crate 能暴露函数给其他 crate 使用. binary crate 为独立运行.

