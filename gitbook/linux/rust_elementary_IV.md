# Rust 基础 IV

错误处理, 泛型, Trait, 生命周期, 测试

## 错误处理

通常 Rust 在编译时会提示错误, 需要程序员处理. 运行时的错误, Rust 没有异常机制, 使用两种错误分类处理

**错误分类**

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



**传播错误** 

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







## Trait







## 生命周期







## 测试





