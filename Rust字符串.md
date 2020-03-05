## Rust为什么会有String和&str？

Rust程序部分：

* print程序
* 错误处理
* 迭代
* 传递字符串转换成大写
* 索引

### print程序

让我们看看实现打印参数，Rust程序是怎样实现的：

```Rust
$ cargo new rustre
     Created binary (application) `rustre` package
$ cd rustre
```

```Rust
fn main() {
    let arg = std::env::args()
        .skip(1)
        .next()
        .expect("should have one argument");
    println!("{}", arg.to_uppercase());
}
```

以上内容的说明：`std::env::args()`返回一个`Iterator`字符串。`skip(1)`忽略程序名称（通常是第一个参数），`next()`获取迭代器中的下一个元素（第一个“实际”）参数。可能有下一个参数，也可能没有。如果没有，`.expect(msg)`通过停止程序打印`msg`。如果有，就有了一个`Option<String>`！

```Rust
$ cargo run
    Finished dev [unoptimized + debuginfo] target(s) in 0.01s
     Running `target/debug/rustre`
thread 'main' panicked at 'should have one argument', src/libcore/option.rs:1188:5
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace.
```

好的！因此，当我们不传递参数时，运行程序会有如上输出。让我们传递一些测试字符串：

```Rust
$ cargo run --quiet -- "noël"
NOËL
$ cargo run --quiet -- "trans rights"
TRANS RIGHTS
$ cargo run --quiet -- "voix ambiguë d'un cœur qui, au zéphyr, préfère les jattes de kiwis"
VOIX AMBIGUË D'UN CŒUR QUI, AU ZÉPHYR, PRÉFÈRE LES JATTES DE KIWIS
$ cargo run --quiet -- "heinz große"
HEINZ GROSSE
```

一切都测试了！最后一个特别酷，用德语的“ß”，确实是“ss”的连字。好吧，这很复杂，但这就是要点。

### 错误处理

因此Rust的行为就像字符串是UTF-8一样，这意味着它必须在某个时刻解码我们的命令行参数，意味着这可能会失败。但是，只在没有参数的情况下看到错误处理，而对于参数无效的UTF-8则看不到错误处理。什么是无效的UTF-8？好吧，我们已经看到“é”被编码为“c3 e9”，所以可以这样工作：

```Rust
$ cargo run --quiet -- $(printf "\\xC3\\xA9")
É
```

我们已经看到一个双字节的UTF-8序列具有：

* 在第一个字节中指示它是一个双字节的序列（前三个位，110）
* 在第二个字节中指示它是多字节序列的延续（前两个位10）

如果我们开始读取一个双字节的序列，然后突然停止怎么办？如果我们传入了C3，但未传入A9呢？

```Rust
$ cargo run --quiet -- $(printf "\\xC3")
thread 'main' panicked at 'called `Result::unwrap()` on an `Err` value: "\xC3"', src/libcore/result.rs:1188:5
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace.
```

查看错误堆栈信息。

```Rust
$ RUST_BACKTRACE=1 cargo run --quiet -- $(printf "\\xC3")
thread 'main' panicked at 'called `Result::unwrap()` on an `Err` value: "\xC3"', src/libcore/result.rs:1188:5                                                
stack backtrace:
(cut)
  13: core::result::unwrap_failed
             at src/libcore/result.rs:1188
  14: core::result::Result<T,E>::unwrap
             at /rustc/5e1a799842ba6ed4a57e91f7ab9435947482f7d8/src/libcore/result.rs:956
  15: <std::env::Args as core::iter::traits::iterator::Iterator>::next::{{closure}}
             at src/libstd/env.rs:789
  16: core::option::Option<T>::map
             at /rustc/5e1a799842ba6ed4a57e91f7ab9435947482f7d8/src/libcore/option.rs:450
  17: <std::env::Args as core::iter::traits::iterator::Iterator>::next
             at src/libstd/env.rs:789
  18: <&mut I as core::iter::traits::iterator::Iterator>::next
             at /rustc/5e1a799842ba6ed4a57e91f7ab9435947482f7d8/src/libcore/iter/traits/iterator.rs:2991
  19: core::iter::traits::iterator::Iterator::nth
             at /rustc/5e1a799842ba6ed4a57e91f7ab9435947482f7d8/src/libcore/iter/traits/iterator.rs:323
  20: <core::iter::adapters::Skip<I> as core::iter::traits::iterator::Iterator>::next
             at /rustc/5e1a799842ba6ed4a57e91f7ab9435947482f7d8/src/libcore/iter/adapters/mod.rs:1657
  21: rustre::main
             at src/main.rs:2
(cut)
```

基本上是这样：

* 在`main()`
* 我们调用`Iterator`的`.next()`
* 最后调用`Result`的`.unwrap()`
* 此时panicked

这意味着只有当我们尝试将参数作为String获取时，它才会出现panic。如果我们将其作为`OsString`，就不会panic：

```Rust
fn main() {
    let arg = std::env::args_os()
        .skip(1)
        .next()
        .expect("should have one argument");
    println!("{:?}", arg)
}
```

```Rust
$ cargo run --quiet -- hello
"hello"
$ cargo run --quiet $(printf "\\xC3")
"\xC3"
```

但是它没有`.to_uppercase()`方法。因为它是一个`OsString`，它是一系列字节。C程序如何处理无效的UTF-8输入？

```C
$ ../upper $(printf "\\xC3")
U+00C0 U+0043 U+0044 U+0050 U+0041 U+0054 U+0048 U+003D U+002E U+003A U+002F U+0068 U+006F U+006D U+0065 U+002F U+0061 U+006D U+006F U+0073 U+002F U+0072 U+0075 U+0073 U+0074 U+003A U+002F U+0068 U+006F U+006D U+0065 U+002F U+0061 U+006D U+006F U+0073 U+002F U+0067 U+006F U+003A U+002F U+0068 U+006F U+006D U+0065 U+002F U+0061 U+006D U+006F U+0073 U+002F U+0066 U+0074 U+006C U+003A U+002F U+0068 U+006F U+006D U+0065 U+002F U+0061 U+006D U+006F U+0073 U+002F U+0070 U+0065 U+0072 U+0073 U+006F U+003A U+002F U+0068 U+006F U+006D U+0065 U+002F U+0061 U+006D U+006F U+0073 U+002F U+0077 U+006F U+0072 U+006B 
ÀCDPATH=.:/HOME/AMOS/RUST:/HOME/AMOS/GO:/HOME/AMOS/FTL:/HOME/AMOS/PERSO:/HOME/AMOS/WORK
```

答案是：不好。实际上一点也不好。UTF-8解码器首先读取C3，然后读取下一个字节（是空终止符），结果应为“à”。但它不再停下来，而是读完参数末尾，直接进入环境块，找到第一个环境变量。现在，在这种情况下，这似乎很温和。但是如果该C程序被用作Web服务器的一部分，并且其输出直接显示给用户怎么办？如果第一个环境变量不是`CDPATH`，而是 `SECRET_API_TOKEN`怎么办？那将是一场灾难。

但如果命令行参数是无效的UTF-8，Rust程序就会尽早panic。如果想优雅地处理这种情况怎么办？可以使用`OsStr::to_str`，它返回一个`Option`值。

```Rust
fn main() {
    let arg = std::env::args_os()
        .skip(1)
        .next()
        .expect("should have one argument");

    match arg.to_str() {
        Some(arg) => println!("valid UTF-8: {}", arg),
        None => println!("not valid UTF-8: {:?}", arg),
    }
}
```

```Rust
$ cargo run --quiet -- "é"
valid UTF-8: é
$ cargo run --quiet -- $(printf "\\xC3")
not valid UTF-8: "\xC3"
```

精彩。我们学到了什么？

在Rust中，只要你不明确地用`unsafe`，类型`String`的值永远是有效的UTF-8。如果尝试使用无效的UTF-8构建`String`，则会出现错误。一些程序，像`std::env::args()`会隐藏错误处理，因为错误的情况非常少。但它仍然会检查错误，并会检查是否发生错误，因为这样做是安全的。

相比之下，C没有字符串类型。它甚至没有真正的字符类型。char是一个ASCII字符加上一个附加位，实际上，它只是一个带符号的8位整数：`int8_t`。绝对不能保证`char *`其中的任何内容都是有效的UTF-8。没有与`char *`关联的编码，只是内存中的地址。也没有关联的长度，计算其长度涉及找到空终止符。空终止字符也是一个严重的安全问题，更不用说NUL是有效的Unicode字符，因此以空字符结尾的字符串不能表示所有有效的UTF-8字符串。

### 迭代 Iteration

我们将如何用空格分隔字符？

```Rust
fn main() {
    let arg = std::env::args()
        .skip(1)
        .next()
        .expect("should have one argument");

    for c in arg.chars() {
        print!("{} ", c);
    }
    println!()
}
```

```Rust
$ cargo run --quiet -- "cup of tea"
c u p   o f   t e a 
```

很简单！让我们尝试使用非ASCII字符：

```Rust
$ cargo run --quiet -- "23€ ≈ ¥2731"
2 3 €   ≈   ¥ 2 7 3 1 
$ cargo run --quiet -- "memory safety 🥺 please 🙏"
m e m o r y   s a f e t y   🥺   p l e a s e   🙏 
```

一切似乎都很好。如果我们要打印Unicode标量值的数字而不是它们的字形，该怎么办？

```Rust
fn main() {
    let arg = std::env::args()
        .skip(1)
        .next()
        .expect("should have one argument");

    for c in arg.chars() {
        print!("{} (U+{:04X}) ", c, c as u32);
    }
    println!()
}
```

```Rust
$ cargo run --quiet -- "aimée"
a (U+0061) i (U+0069) m (U+006D) é (U+00E9) e (U+0065)
```

酷！如果我们想显示其为UTF-8编码怎么办？我的意思是打印单个字节？

```Rust
fn main() {
    let arg = std::env::args()
        .skip(1)
        .next()
        .expect("should have one argument");

    for b in arg.bytes() {
        print!("{:02X} ", b);
    }
    println!()
}
```

```Rust
$ cargo run --quiet -- "aimée"
61 69 6D C3 A9 65 
```

有我们的"c3 a9"！很简单。目前为止，我们还没对类型的担心，在我们的Rust程序中还没有一个`String`或`&str`。所以，让我们去寻找麻烦。

### 传递字符串转换成大写

```Rust
fn main() {
    let arg = std::env::args()
        .skip(1)
        .next()
        .expect("should have one argument");

    println!("upp = {}", uppercase(arg));
    println!("arg = {}", arg);
}

fn uppercase(s: String) -> String {
    s.to_uppercase()
}
```

```Rust
$ cargo build --quiet
error[E0382]: borrow of moved value: `arg`
 --> src/main.rs:8:26
  |
2 |     let arg = std::env::args()
  |         --- move occurs because `arg` has type `std::string::String`, which does not implement the `Copy` trait
...
7 |     println!("upp = {}", uppercase(arg));
  |                                    --- value moved here
8 |     println!("arg = {}", arg);
  |                          ^^^ value borrowed here after move

error: aborting due to previous error

For more information about this error, try `rustc --explain E0382`.
error: could not compile `rustre`.
```

哦，上帝，编译器来了。问题在于我们将arg传入`uppercase()`，然后又再次使用它。我们可以先打印arg，然后再调用uppercase()。那行得通吗？可以。但是，假设我们就是需要先调用`uppercase`呢？

```Rust
fn main() {
    let arg = std::env::args()
        .skip(1)
        .next()
        .expect("should have one argument");

    println!("upp = {}", uppercase(arg.clone()));
    println!("arg = {}", arg);
}

fn uppercase(s: String) -> String {
    s.to_uppercase()
}
```

```Rust
$ cargo run --quiet -- "dog"
upp = DOG
arg = dog
```

但是这有点愚蠢。为什么我们需要克隆arg？只是传入`uppercase`，我们不需要在内存中有第二个拷贝。现在在内存中，我们有：

* arg（“dog”）
* arg的拷贝，我们传入`uppercase()`（“dog”）
* uppercase()返回值（“DOG”）

我猜这是`&str`存在的意义吧？让我们尝试一下：

```Rust
fn main() {
    let arg = std::env::args()
        .skip(1)
        .next()
        .expect("should have one argument");

    println!("upp = {}", uppercase(arg));
    println!("arg = {}", arg);
}

fn uppercase(s: &str) -> String {
    s.to_uppercase()
}
```

```Rust
cargo run --quiet -- "dog"
error[E0308]: mismatched types
 --> src/main.rs:7:36
  |
7 |     println!("upp = {}", uppercase(arg));
  |                                    ^^^
  |                                    |
  |                                    expected `&str`, found struct `std::string::String`
  |                                    help: consider borrowing here: `&arg`
```

根据编译器的提示修改：

```Rust
println!("upp = {}", uppercase(&arg));
```

```Rust
$ cargo run --quiet -- "dog"
upp = DOG
arg = dog
```

为了使其更接近于C代码，我们应该：

* 分配一个“目标”
* 传递“目标”到`uppercase()`
* `uppercase()`遍历每个字符，将其转换为大写，并将其附加到"目标"

```Rust
fn main() {
    let arg = std::env::args()
        .skip(1)
        .next()
        .expect("should have one argument");

    let mut upp = String::new();
    println!("upp = {}", uppercase(&arg, upp));
    println!("arg = {}", arg);
}

fn uppercase(src: &str, dst: String) -> String {
    for c in src.chars() {
        dst.push(c.to_uppercase());
    }
    dst
}
```

```Rust
$ cargo run --quiet -- "dog"
error[E0308]: mismatched types
  --> src/main.rs:14:18
   |
14 |         dst.push(c.to_uppercase());
   |                  ^^^^^^^^^^^^^^^^ expected `char`, found struct `std::char::ToUppercase`
```

>ToUppercase，该结构由char上的to_uppercase方法创建，返回一个迭代器，该迭代器生成char的大写等效项。

迭代器，知道这一点，我们可以使用`for x in y`：

```Rust
fn uppercase(src: &str, dst: String) -> String {
    for c in src.chars() {
        for c in c.to_uppercase() {
            dst.push(c);
        }
    }
    dst
}
```

```Rust
$ error[E0596]: cannot borrow `dst` as mutable, as it is not declared as mutable
  --> src/main.rs:15:13
   |
12 | fn uppercase(src: &str, dst: String) -> String {
   |                         --- help: consider changing this to be mutable: `mut dst`
...
15 |             dst.push(c);
   |             ^^^ cannot borrow as mutable
```

让我们看一下`String::push`的声明：

```Rust
pub fn push(&mut self, ch: char)
```

因此`dst.push(c)`与`String::push(&mut dst, c)`完全相同。根据编译器建议修改：

```Rust
fn uppercase(src: &str, mut dst: String) -> String {
	...
}
```

```Rust
$ cargo run --quiet -- "dog"
upp = DOG
arg = dog
```

`uppercase`没有返回值呢？

```Rust
fn uppercase(src: &str, mut dst: String) {
    for c in src.chars() {
        for c in c.to_uppercase() {
            dst.push(c);
        }
    }
}
```

```Rust
cargo run --quiet -- "dog"
error[E0382]: borrow of moved value: `upp`
  --> src/main.rs:10:26
   |
7  |     let upp = String::new();
   |         --- move occurs because `upp` has type `std::string::String`, which does not implement the `Copy` trait
8  |     uppercase(&arg, upp);
   |                     --- value moved here
9  | 
10 |     println!("upp = {}", upp);
   |                          ^^^ value borrowed here after move
```

我们需要让upp可变地借用。

```Rust
fn main() {
    let arg = std::env::args()
        .skip(1)
        .next()
        .expect("should have one argument");

    let mut upp = String::new();
    // was just `upp`
    uppercase(&arg, &mut upp);

    println!("upp = {}", upp);
    println!("arg = {}", arg);
}

// was `mut dst: String`
fn uppercase(src: &str, dst: &mut String) {
    for c in src.chars() {
        for c in c.to_uppercase() {
            dst.push(c);
        }
    }
}
```

```Rust
$ cargo run --quiet -- "dog"
upp = DOG
arg = dog
```

现在又可以使用了！可增长的字符串，这是否意味着我们可以预分配合理大小的String，然后将其重新用于多个uppercase 调用？

### 索引

C允许我们直接索引，Rust允许我们这样做吗？

```Rust
fn main() {
    for arg in std::env::args().skip(1) {
        for i in 0..arg.len() {
            println!("arg[{}] = {}", i, arg[i]);
        }
    }
}
```

```Rust
$ cargo run --quiet -- "dog"
error[E0277]: the type `std::string::String` cannot be indexed by `usize`
 --> src/main.rs:4:41
  |
4 |             println!("arg[{}] = {}", i, arg[i]);
  |                                         ^^^^^^ `std::string::String` cannot be indexed by `usize`
  |
  = help: the trait `std::ops::Index<usize>` is not implemented for `std::string::String`
```

我们不可以。我们可以先将其转换为Unicode标量值数组，然后对其进行索引：

```Rust
fn main() {
    for arg in std::env::args().skip(1) {
        let scalars: Vec<char> = arg.chars().collect();
        for i in 0..scalars.len() {
            println!("arg[{}] = {}", i, scalars[i]);
        }
    }
}
```

```Rust
$ cargo run --quiet -- "dog"
arg[0] = d
arg[1] = o
arg[2] = g
```

是的，行得通！老实说，这样比较好，因为我们只需要解码一次UTF-8字符串，然后我们就可以进行随机访问了。这可能就是为什么它没有实现`Index<usize>`的原因。

有趣的事情：`Index<Range<usize>>`。

```Rust
fn main() {
    for arg in std::env::args().skip(1) {
        let mut stripped = &arg[..];
        while stripped.starts_with(" ") {
            stripped = &stripped[1..]
        }
        while stripped.ends_with(" ") {
            stripped = &stripped[..stripped.len() - 1]
        }
        println!("     arg = {:?}", arg);
        println!("stripped = {:?}", stripped);
    }
}
```

```Rust
$ cargo run --quiet -- "  floating in space   "
     arg = "  floating in space   "
stripped = "floating in space"
```

>String是堆分配的，因为它是可增长的。而&str可以从任何地方引用数据：堆，栈，甚至程序的数据段。

`&str`，它是不同的，它指向相同的内存区域，只是在不同的偏移量处开始和结束。实际上，我们可以使其成为一个函数：

```Rust
fn main() {
    for arg in std::env::args().skip(1) {
        let stripped = strip(&arg);
        println!("     arg = {:?}", arg);
        println!("stripped = {:?}", stripped);
    }
}

fn strip(src: &str) -> &str {
    let mut dst = &src[..];
    while dst.starts_with(" ") {
        dst = &dst[1..]
    }
    while dst.ends_with(" ") {
        dst = &dst[..dst.len() - 1]
    }
    dst
}
```

而且效果也一样。不过，这似乎很危险。如果原始字符串的内存被释放怎么办？

```Rust
fn main() {
    let stripped;
    {
        let original = String::from("  floating in space  ");
        stripped = strip(&original);
    }
    println!("stripped = {:?}", stripped);
}
```

```Rust
$ cargo run --quiet -- "  floating in space   "
error[E0597]: `original` does not live long enough
 --> src/main.rs:5:26
  |
5 |         stripped = strip(&original);
  |                          ^^^^^^^^^ borrowed value does not live long enough
6 |     }
  |     - `original` dropped here while still borrowed
7 |     println!("stripped = {:?}", stripped);
  |                                 -------- borrow later used here
```

在Rust中？编译器将检查所有的"恶作剧"。

最后，String用范围索引，很酷，但是`..`是字符范围吗？

```Rust
fn main() {
    for arg in std::env::args().skip(1) {
        println!("first four = {:?}", &arg[..4]);
    }
}
```

```Rust
$ cargo run --quiet -- "want safety?"
first four = "want"
$ cargo run --quiet -- "🙈🙉🙊💥"
first four = "🙈"
```

字节范围。我以为所有Rust字符串都是UTF-8？但是使用切片，我们可以得到部分多字节序列，或无效的UTF-8？假如：

```Rust
fn main() {
    for arg in std::env::args().skip(1) {
        println!("first two = {:?}", &arg[..2]);
    }
}
```

```Rust
$ cargo run --quiet -- "🙈🙉🙊💥"
thread 'main' panicked at 'byte index 2 is not a char boundary; it is inside '🙈' (bytes 0..4) of `🙈🙉🙊💥`', src/libcore/str/mod.rs:2069:5
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace.
```

那太好了。它会panic，这是安全的事情。

### 结束语

无论如何，这篇文章已经很长了。希望它对Rust中的字符串处理有足够的介绍，以及Rust为什么同时具有String和&str。

答案当然依旧是安全性，正确性和性能。

在我们编写的最后一个Rust字符串操作程序时，确实遇到了障碍，但是它们主要是编译时错误或panic。我们没有一次：

* 从无效地址读取
* 写入无效的地址
* 忘了释放东西
* 覆盖了其他一些数据
* 需要一个额外的工具来告诉我们问题出在哪里

而且，加上令人惊叹的编译器信息以及社区，这就是Rust的美。






