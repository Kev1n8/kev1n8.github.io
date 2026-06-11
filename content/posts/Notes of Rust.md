+++
title = "Notes of Rust"
date = 2023-04-24

[taxonomies]
tags = ["rust"]
+++



## 可变和不可变变量

`let mut, let `

`const` 与 `let`无`mut`区别在于，`let`有`ownership`，而`const`随便使用

## 数据类型

### `Scalar`

`integer, float, boolean, character`

### `Compound`

`tuple., array[]`

## 函数
```rust  
fn func_name(para:type) -> return_type{}
```

## 注释
## 控制流

```rust
if,else if,else,loop,break,while,for,..(=)
```
## 所有权

会转让或释放所有权的情况（储存堆上的变量）：
=移动、函数传参数、所在域结束
禁止发生的情况：
在当前域有两个只向同一变量的可变引用、
在当前域内同时有可变引用和不可变引用并且在可变引用后再次使用了之前的不可变引用、
代码块返回一个生命已经终结的引用、
可变引用会触发`borrowing`，引用指向的变量失去改变自己的权利，在引用死亡前禁止变量改变自己、
引用指向变量生命必须比引用本身长


## 结构
```rust
struct Struct_name{
    data_name:type_name
}
let v = Struct_name{//赋值};
let v = Struct_name{..struct1};
fn build_struct(name:type_name)->{
    name,
}
//note that all operation above takes the
//ownership from the parameters
struct Point(i32,i32,i32);
```

实现结构内的方法：`impl,self`

## 枚举
```rust
enum Kind{
    k1,
    k2,
}
```

访问方式::

`varient`可以附带有数据类型`()`互相可以不一样。

```rust
enum Message{
    Quit,//no data associated
    Move{x:i32,y:i32},//has a name fields, like a struct
    Write(String),// has a String
    ChangeColor(i32,i32,i32),// includes 3 value, like a tuple
}

match enum_type{
    ::case => return_value,
    _ => return_else_value,
}

if let //for sequence only contains 2 branches
if let Some(num) = num_type_is_Option{
    println!("{num}");
}
else{
    println!("not a num");
}
//match 必须包含所有branches

//Rust 内置Option枚举
enum Option<T>{
    Some(T),
    None,
}
```
## Packages, Crates and Modules

调用`cargo new`会创建一个新项目，就是一个`Package`
    
```rust
Cargo.toml
src
    main.rs
```

`Crate`有两种类型：`bin`和`lib`
    

```rust
src/main.rs //bin类型的Crate
src/lib.rs //lib类型的Crate
一个package中的Crate可以有多个bin类型，最多只能有一个lib类型，
当有多个bin，需要放到src/bin下。

bin类型use的path不能使用crate开头，必须是Crate本名。

re-exporting： 利用pub修饰use，让导入当前scope的第三者可以引用当前scope已经use的item
```

`module:`在Crate内组织代码

## 常用的`Collections`

### `vectors`

```rust
let v = vec![1,2,3];
//let v: Vec<i32> = Vec::new();
v.push(4);//mutable reference
v.get(2);//return an Option
```

### `String and UTF-8`
use chars() or bytes() to visit item in side string

```rust
s = String::from("hello");
//for i in "hello".chars()
for i in s.chars(){
    println!("{i}");
}
```

### `HashMap`
```rust
use std::collections::HashMap;

let mut scores = HashMap::new();

scores.insert("Blue".to_string(),10);
//insert会夺走ownership

let team_name = String::from("Blue");
let score = score.get(&team_name).copied().unwrap_or(0);
//get()返回一个Option<&V>，copied返回Option<&V>的一个拷贝，得到
//ownership，用unwrap_or()“解包”返回一个i32

for (k,v) in &scores{
    println!("{k}:{v}");
}

overwriting: insert()
adding: entry().or_insert()
//entry()return &mut to the value and use or_insert() to insert
```

## Error Handling

## 统称
可用于`fn,struct,enum`
e.g. 

```rust
fn example<T>(pa:&T)->&T{
    do stuff
}
struct<T,Y>{
    name:T,
    gender:Y,
}
enum E<T>{
    name(T),
}

fn example<T: some_trait>(pa: &T){
    //pass
}
```

## 特点

### 特点的声明：

```rust
pub trait Summary{
    fn summarize(&self)->String;
}
```

### 使用关键词`for`赋予结构特点，例子给两个不同的结构体赋予`Summary`的特点：

```rust
pub struct NewsArticle {
    pub headline: String,
    pub location: String,
    pub author: String,
    pub content: String,
}

impl Summary for NewsArticle {
    fn summarize(&self) -> String {
    format!("{}, by {} ({})", self.headline,self.author,self.location)
    }
}

pub struct Tweet {
    pub username: String,
    pub content: String,
}

impl Summary for Tweet {
    fn summarize(&self) -> String {
        format!("{}: {}", self.username, self.content)
    }
}
```

### 调用方式，类似函数：
这里use还要包括`Summary`，因为`trait`是储存在块本地的；
另外，已经定义的结构和特点不能重复定义

```rust
use aggregator::{Summary, Tweet};

fn main() {
    let tweet = Tweet {
        username: String::from("horse_ebooks"),
        content: String::from("of course, as you probably already know, people",
        ),
        reply: false,
        retweet: false,
    };
    println!("1 new tweet: {}", tweet.summarize());
}
```
### 可以设置默认的特点：

```rust
pub trait Summary{
    fn summarize(&self) -> String{
        String::from("Read more...")
    }
}

impl Summary for NewsArticle {} //必须有这句话，而且括号内没有对应的方法
```

### 把特点作为函数参数：
把特点作为参数的意义在于，任何实现了这个特点的数据类型都能作为这个函数的参数

```rust
pub fn notify(item: &impl Summary){
    println!("Breaking news! {}",item.summarize());
}

pub fn notify<T: Summary>(item: T){//等价
    println!("Breaking news! {}",item.summarize());
}
```

### 同时满足多个特点：

```rust
pub fn notify(item: &(impl Summary + Display)) {}

pub fn notify<T: Summary + Display>(item: &T){}
```

### 更多例子：

```rust
fn func<T: Display + Clone, U: Clone + Debug>(t: &T, u: &U){}

fn func<T,U>(t: &T, u: &U)
where
    T: Display + Clone,
    U: Clone + Denug,
{}
```

### 要求返回值实现特点：

```rust
fn returns_summarizable() -> impl Summary {
    Article{
        name:"Jane".to_string,
        published:false,
    }
}
```

### 实现包含泛型的结构方法时，区别对待：

```rust
struct Pair<T> {
    x:T,
    y:T,
}

impl<T> Pair<T> {
    fn new(x:T, y:T)->Self{
        Self { x, y }
    }
}

impl<T: Display + ParialOrd> Pari<T> {
    fn cmp_display(&self) {
        if self.x >= self.y {
            ...
        }
        else {
            ...
        }
    }
}
```

## `Lifetime`
基本的在之前已经提到过

### `Borrow Checker`

```rust
fn main() {
    let r;                // ---------+-- 'a
    {                     //          |
        let x = 5;        // -+-- 'b  |
        r = &x;           //  |       |
    }                     // -+       |
    println!("r: {}", r); //          |
}                         // ---------+

fn main() {
    let x = 5;            // ----------+-- 'b
    let r = &x;           // --+-- 'a  |
    println!("r: {}", r); // --+       |
}                         // ----------+
```

### 参数为引用编译器无法判断生命周期时

```rust
fn longest(x: &str, y: &str) -> &str {
    if x.len() > y.len() {
        x
    } else {
        y
    }
}

fn longest<'a>(x: &'a str, y: &'a str) -> &'a str{}
// 参数和返回值具有相同的生命周期'a 
// 返回值的lifetime = min(x,y)
```

之所以参数都要需要标记，是因为`x`，`y`都有可能被返回

```rust
fn longest<'a>(x: &'a str, y: &str) -> &'a str {
    x
}
```

### 结构体内的变量也可以是引用，但是必须指明生命周期

```rust
struct SSS<'a>{
    a_str:&'a str,
}//意味着，一个SSS实例的lifetime不能大于a_str

fn main(){
    let sss = SSS{
        a_str:{
            let tmp = "hello".to_string();//生命周期过短
            tmp.as_str()
        },
        //a_str:"hello",//正常运行
    };
    println!("{}",sss.a_str);
}
```

### 无需声明生命周期的情况(`lifetime elision rules`)

编译器检查生命周期的三条规则，一旦能确认所有参数和输出值的生命周期，编译器不强制要求添加`lifetime annotation`：

1. 给每个参数赋予不同的生命周期
2. 只有一个参数时，返回值的生命周期和这个参数一样
3. 当参数形式为`&self`时，因为`self`，所以所有生命周期都是`self`的生命周期

    ```rust
    fn first_word(s: &str) -> &str{//rule 1
    fn first_word<'a>(s: &'a str) -> &str{//rule 2
    fn first_word<'a>(s: &'a str) -> &'a str{//rule 3
    
    fn longest(x: &str, y: &str) -> &str {//rule1
    fn longest<'a, 'b>(x: &'a str, y: &'b str) -> &str {//rule2
    ```

## 泛型、特点、生命周期大杂烩

```rust
use std::fmt::Display;

fn longest_with_an_announcement<'a, T>(
   x: &'a str,
    y: &'a str,
    ann: T,
) -> &'a str
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

### 三个测试函数

在`Rust`中，可以为自己编写的程序编写测试以测试代码的正确性。
在终端调用`cargo new proj_name --lib`会自动生成项目，`src`文件夹的`lib.rs`会自带`test module`。

```rust
#[cfg(test)]
mod tests {
    #[test]
    fn it_works() {
        let result = 2 + 2;
        assert_eq!(result, 4);
    }
}
```
其中`#[tests]`用来指示接下来的函数是用于测试的函数，`assert_eq!()`判断参数是否相同，当结果不同或者代码发生`panic!`就会测试失败，另`assert_ne!()`则判断是否不同。

`assert!`参数类型是`Boolean`，如果参数为`false`则测试失败。

### PartialEq Debug 特点

如果一个结构体或者枚举想要作为参数传入上述函数，必须实现`PartialEq`和`Debug`这两个特点，（前者在于`==、!=`的比较，后者在于代码判断为`Failed`时要打印错误信息）

### 自定义失败信息

`assert`函数第二个参数即为自定义字符串信息：
    

```rust
pub fn greeting(name: &str) -> String {
    String::from("Hello!")
}
#[cfg(test)]
mod tests {
    use super::*;
    #[test]
    fn greeting_contains_name() {
        let result = greeting("Carol");
        assert!(
            result.contains("Carol"),
            "Greeting did not contain name, value was `{}`",
            result
        );
    }
}
```

报错信息则为`"Greeting did not contain name, value was 'Carol'"`。

### should_panic
在`#[test]`属性下一行添加`#[should_panic]`，让编译器认为检测函数应该`panic`，否则测试不通过。
    
```rust
#[test]
#[should_panic]
fn example() {
    panic!("This is supposed to panic.");//测试ok
}
```

使用`expected`可以使`should_panic`更加精确：
例如`#[should_panic(expected = "is supposed")]`替换上面的`#[should_panic]`，
则测试可以通过，因为`panic contained expected string`。

### Result<T, E>
在`test module`中，可以使用`Result`枚举来代替`assert`系列函数：

```rust
#[cfg(test)]
mod tests {
    #[test]
    fn it_works() -> Result<(), String> {
        let value = 
        if 2 + 2 == 4 {
            Ok(())
        } else {
            Err(String::from("two plus two does not equal four"))
        };
        value
    }
}
```

值得注意的是，在这种情况下不能使用`should_panic`，非要使用的话，可以在函数最后`assert!(value.is_err())`;

### 控制测试方式
`cargo test`

1. `-- --test-thread=1 `线程设置为`1`，禁止测试函数并行运行
2. `-- --show-output `就算测试不通过也打印过程中`println`的信息
3. `cargo test prefix*` 只测试以`prefix`打头的函数
4. 在代码中`#[test]`下一行添加`#[ignore]`可以使测试忽略当前函数
5. `-- --ignored` 只运行标记了`ignore`的函数

### 单元测试和集成测试
单元测试：
更小更集中，在隔离的环境中测试一个模块，上面的例子都是单元测试；
使用`#[cfg(test)]`标注，只在`cargo test`时编译和运行

集成测试：
在外部的，位于与一般代码相同的文件中。需要新建与`src`同级的`tests`目录，在目录下新建`test_name.rs`，需要使用`use`导入测试目标`crate`，代码无需`cfg`标记，只需在每个测试函数前`#[test]`即可。

## 闭包

任何花括号包括的区域最后一条语句没有加分号都有返回值，{}闭包就是可以保存进变量 或 作为参数传递给其他函数 的 匿名函数。
函数变量e.g.

```rust
fn  add_one_v1   (x: u32) -> u32 { x + 1 }
let add_one_v2 = |x: u32| -> u32 { x + 1 };
let add_one_v3 = |x|             { x + 1 };
let add_one_v4 = |x|               x + 1  ;
```

## 迭代器

### Iterator & next

```rust
pub trait Iterator{
  type Item;
  
  fn next(&mut self) -> Option<Self::Item>;
}
```

如果直接把迭代器 `iter()` 赋值给变量 `v1`，那么在直接调用 `v1.next()` 要求 `v1 `是可变的，如果是 `for` 来遍历则不需要，因为 `rust `后台自动获取 `v1` 的所有权并使它可变。

需要注意，`vector  `迭代器的 `next ` 返回的是不可变引用，如果需要得到所有权，可以调用`into_iter`；如果需要可变迭代引用，可以调用 `iter_mut`。

### 消费迭代器的方法

`Iterator trait `中调用了 `next `的方法被称为**消费适配器**( `consuming adaptors` )，这些方法会消耗迭代器，夺走所有权。

以sum为例：

```rust
#[test]
fn iterator_sum() {
    let v1 = vec![1, 2, 3];

    let v1_iter = v1.iter();

    let total: i32 = v1_iter.sum();

    assert_eq!(total, 6);
}
```

这里调用 `sum` 之后不再允许使用 `v1_iter`，因为所有权用完了。

### 简单编写自定义next

```rust
struct cnt{
  	count:u32,
}

impl cnt{
 		fn new() -> cnt{
    		cnt{count:0}
  	}
}

impl Iterator for cnt{
  	type Item = u32;
  	fn next(&mut self) -> Option<Self::Item>{
      	self.count += 1;
    		if self.count < 6 {
      			Some(self.count)
  			} else {
      			None
      	}
  	}
}
```

这样，就可以获得可以`next`六次的迭代器了。

## 循环VS迭代器

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

为了计算 `prediction` 的值，这些代码遍历了 `coefficients` 中的 12 个值，使用 `zip` 方法将系数与 `buffer` 的前 12 个值组合在一起。接着将每一对值相乘，再将所有结果相加，然后将总和右移 `qlp_shift` 位。

这里，我们创建了一个迭代器，使用了两个适配器，接着消费了其值。Rust 代码将会被编译为什么样的汇编代码呢？遍历 `coefficients` 的值完全用不到循环：Rust 知道这里会迭代 12 次，所以它“展开”（unroll）了循环。展开是一种移除循环控制代码的开销并替换为每个迭代中的重复代码的优化。

所有的系数都被储存在了寄存器中，这意味着访问他们非常快。这里也没有运行时数组访问边界检查。所有这些 Rust 能够提供的优化使得结果代码极为高效。

## 智能指针

其实，应用最多的指针是 `&` 引用，但是Rust中有一类结构实现了 `Deref` 和 `Drop` trait，前者让结构体实例能像引用那样（可以“解包”），后者可以自定义当智能指针离开作用域要运行的代码。

其实，`String  ` 和 `Vec<T>` 都属于智能指针。

### `Box<T>`

在堆上储存数据，在栈上储存指针。

#### 递归枚举

如果要自定义一个枚举列表：

```rust
enum List {
    Cons(i32, List),
    Nil,
}
```

上面的代码是不能编译的，因为List中永远有下一个List，而每个List都附带一个i32（在栈上），编译器不知道具体要分配多少空间。

![An infinite Cons list](/images/trpl15-01.svg)

```rust
enum List {
    Cons(i32, Box<List>),
    Nil,
}

use crate::List::{Cons, Nil};

fn main() {
    let list = Cons(1,
        Box::new(Cons(2,
            Box::new(Cons(3,
                Box::new(Nil))))));
}
```

上面的代码就是可行的![A finite Cons list](/images/trpl15-02.svg)

### 自定义一个智能指针

#### 实现Deref trait

```rust
use std::ops::Deref;

struct MyBox<T>(T);

impl<T> MyBox<T> {
  	fn new(x:T) -> MyBox<T> {
      	MyBox(x)
    }
}

impl<T> Deref for MyBox<T> {
    type Target = T;

    fn deref(&self) -> &T {
        &self.0
    }
}
```

以上代码能够让 `MyBox` 能够像 `dereference` 引用 `&` 一样解包 `MyBox` 实例，这里返回的是 `&x`，因此实际Rust在底层运行的代码是：

```rust
*(y.deref())//y是一个MyBox实例，deref()返回的是引用
```

### 借助Deref trait实现隐式解引用强制转换

有这样一个函数：

```rust
fn hello(name: &str) {
    println!("Hello, {}!", name);
}
```

要求参数是 `&str`，而：

```rust
fn main() {
    let m = MyBox::new(String::from("Rust"));
    hello(&m);
}
```

这段代码却可以正常运行，这是因为 `MyBox` 和 `String` 都有实现 `Deref` trait，这里使用 `&m` 调用 `hello` 函数，其为 `MyBox<String>` 值的引用。 Rust 可以通过 `deref` 调用将 `&MyBox<String>` 变为 `&String`。Rust 再次调用 `deref` 将 `&String` 变为 `&str`，这就符合 `hello` 函数的定义了。

如果没有实现解引用强制转换，代码将必须写成这样：

```rust
fn main() {
    let m = MyBox::new(String::from("Rust"));
    hello(&(*m)[..]);
}
```

以上解析都是发生在编译时，不会有运行损耗。

### Drop Trait

之所以把 `Drop` trait 放在智能指针这里，是因为 `Drop` 几乎都是用于实现智能指针。

可以直接把 `Drop` 里的 `drop()` 理解为析构函数，我们不能直接调用 `drop` 因为这样的话作用域结束后又会调用一次，如果需要提前释放变量可以使用 `std::mem::drop` 这个函数不同于前者，位于 prelude 中，通过调用它可以提前释放变量。无需担心之后发生“空指针”的问题，因为编译起会帮你检测，确保引用总是有效的。

### `Rc<T>` 引用计数智能指针

Rust 内置一个叫做 `Rc<T>` 的类型，是 `reference counting` 的缩写。

`Rc<T>` 用于当我们希望在堆上分配内存给程序多个部分读取，而且无法在编译时确定程序的哪一部分会最后结束使用它的时候。如果确实知道哪一部分是最后结束使用的话，就能令其成为数据的所有者，正常的所有权规则就可以在编译时运行。

`RC<T>` 只能用于单线程场景。

类似浅拷贝，不会完整拷贝 `T` 所有数据，而是让**引用计数**增加。在场景：多个列表需要共享一段列表时，不会有所有权“不够用”或者深拷贝低效率的问题。

```rust
enum List {
    Cons(i32, Rc<List>),
    Nil,
}

use crate::List::{Cons, Nil};
use std::rc::Rc;

fn main() {
    let a = Rc::new(Cons(5, Rc::new(Cons(10, Rc::new(Nil)))));
    println!("count after creating a = {}", Rc::strong_count(&a));
    let b = Cons(3, Rc::clone(&a));
    println!("count after creating b = {}", Rc::strong_count(&a));
    {
        let c = Cons(4, Rc::clone(&a));
        println!("count after creating c = {}", Rc::strong_count(&a));
    }
    println!("count after c goes out of scope = {}", Rc::strong_count(&a));
}
OUTPUT:
count after creating a = 1
count after creating b = 2
count after creating c = 3
count after c goes out of scope = 2
```

 `Rc<T>` 允许程序多个部分只读共享数据。

### `RefCell<T>` 和内部可变性模式

**内部可变性**（*Interior mutability*）是 Rust 中的一个设计模式，允许你在有不可变量引用存在的情况下也能改变数据。使用 `unsafe` 来模糊 Rust 通常的可变性和借用规则。

- `RefCell<T>` 代表其数据唯一所有权。

- `RefCell<T>` 用于当你确定代码是正确的，但是编译器不这么认为的时候。

- `RefCell<T>` 同样只能用于单线程场景。

如下为选择 `Box<T>`，`Rc<T>` 或 `RefCell<T>` 的理由：

- `Rc<T>` 允许相同数据有多个所有者；`Box<T>` 和 `RefCell<T>` 有单一所有者。
- `Box<T>` 允许在编译时执行不可变或可变借用检查；`Rc<T>`仅允许在编译时执行不可变借用检查；`RefCell<T>` 允许在运行时执行不可变或可变借用检查。
- 因为 `RefCell<T>` 允许在运行时执行可变借用检查，所以我们可以在即便 `RefCell<T>` 自身是不可变的情况下修改其内部的值。

假如现在有一个结构 `A` ，结构 `B` 内的变量有 `&A` ，一般来说，我们是不能改变 `B` 的，因为它不是可变引用，但是如果 `B` 里面的某个变量是 `RefCell<T>` ，那么就可以调用 `B.some_variable.borrow_mut()` ，这会返回一个可变引用在**内部**实现数据更改。

实例代码：

```rust
pub trait Messenger {
    fn send(&self, msg: &str);
}

pub struct LimitTracker<'a, T: Messenger> {
    messenger: &'a T,
    value: usize,
    max: usize,
}

impl<'a, T> LimitTracker<'a, T>
    where T: Messenger {
    pub fn new(messenger: &T, max: usize) -> LimitTracker<T> {
        LimitTracker {
            messenger,
            value: 0,
            max,
        }
    }

    pub fn set_value(&mut self, value: usize) {
        self.value = value;

        let percentage_of_max = self.value as f64 / self.max as f64;

        if percentage_of_max >= 1.0 {
            self.messenger.send("Error: You are over your quota!");
        } else if percentage_of_max >= 0.9 {
             self.messenger.send("Urgent warning: You've used up over 90% of your quota!");
        } else if percentage_of_max >= 0.75 {
            self.messenger.send("Warning: You've used up over 75% of your quota!");
        }
    }
}

#[cfg(test)]
mod tests {
    use super::*;
    use std::cell::RefCell;

    struct MockMessenger {
        sent_messages: RefCell<Vec<String>>,
    }

    impl MockMessenger {
        fn new() -> MockMessenger {
            MockMessenger { sent_messages: RefCell::new(vec![]) }
        }
    }

    impl Messenger for MockMessenger {
        fn send(&self, message: &str) {
            self.sent_messages.borrow_mut().push(String::from(message));
        }
    }

    #[test]
    fn it_sends_an_over_75_percent_warning_message() {
        // --snip--
        let mock_messenger = MockMessenger::new();
        let mut limit_tracker = LimitTracker::new(&mock_messenger, 100);
        limit_tracker.set_value(75);

        assert_eq!(mock_messenger.sent_messages.borrow().len(), 1);
    }
}
fn main() {}
```

#### 原理

`borrow` 返回 `Ref<T>`，而 `borrow_mut` 则返回 `RefMut<T>` 二者都是智能指针。类似的，`RefCell` 只允许有多个不可变借用或一个可变借用。如果违反规则的话，相比编译时出错， `RefCell<T>` 会在运行时 panic。

### `Rc<T>` 和 `RefCell<T>`搭配使用

- `Rc<T>` 允许统一数据有多个所有者但不可变引用
- `RefCell` 可以返回可变引用修改不可变引用的值

由此，如果 `Rc<RefCell<T>>` 这样存放数据，就可以得到多个所有者**并且**可以修改的值了。

例子：

```rust
#[derive(Debug)]
enum List {
    Cons(Rc<RefCell<i32>>, Rc<List>),
    Nil,
}

use crate::List::{Cons, Nil};
use std::rc::Rc;
use std::cell::RefCell;

fn main() {
    let value = Rc::new(RefCell::new(5));

    let a = Rc::new(Cons(Rc::clone(&value), Rc::new(Nil)));

    let b = Cons(Rc::new(RefCell::new(6)), Rc::clone(&a));
    let c = Cons(Rc::new(RefCell::new(10)), Rc::clone(&a));

    *value.borrow_mut() += 10;
    // 这里*解引用得到 RefCell，然后对其调用.borrow_mut

    println!("a after = {:?}", a);
    println!("b after = {:?}", b);
    println!("c after = {:?}", c);
}
```

### 利用 `thread::spawn` 实现多线程

结合闭包和 `spawn` 函数产生子线程：

```rust
use std::thread;

fn main() {
    let v = vec![1, 2, 3];
    let a = 'a';

    let handle = thread::spawn(move || {//转移所有权
        println!("Here's a vector: {:?}", v);
        println!("Here's a character: {:?}", a);
    });
    print!("Here's a character: {:?}", a);
    //println!("Here's a vector: {:?}", v);会报错，无所有权

    handle.join().unwrap();//等待子线程结束
}
```

### 利用 `mpsc::channel` 创建通道用于进程间通讯

`mpsc` 的含义是 *mutiple producer, single consumer*。

```rust
use std::thread;
use std::sync::mpsc;

fn main() {
    let (tx, rx) = mpsc::channel();

    thread::spawn(move || {
        let val = String::from("hi");
        tx.send(val).unwrap();
    });

    let received = rx.recv().unwrap();
  //rx.try_recv()尝试接收但是不阻塞
    println!("Got: {}", received);
}
```

使用 `send` 发送讯息，同样送走了变量的所有权（堆）。

可以将 `rx` 作为一个迭代器使用：

```rust
use std::thread;
use std::sync::mpsc;
use std::time::Duration;

fn main() {
    let (tx, rx) = mpsc::channel();

    thread::spawn(move || {
        let vals = vec![
            String::from("hi"),
            String::from("from"),
            String::from("the"),
            String::from("thread"),
        ];

        for val in vals {
            tx.send(val).unwrap();
            thread::sleep(Duration::from_secs(1));
        }
    });

    for received in rx {//这里！阻塞地等待
        println!("Got: {}", received);
    }
}
```

如果要实现多个生产者端口，可以使用 `clone` 得到多个 `tx`。

### `Mutex<T>` 互斥器

单线程 `Mutex` 的一个简单实例：

```rust
use std::sync::Mutex;

fn main() {
    let m = Mutex::new(5);//创建Mutex
    {
        let mut num = m.lock().unwrap();//返回Mutex内的可变引用
        *num = 6;
    }
    println!("m = {:?}", m);//将发现m内为6
}
```

如果多个线程都要使用一个 `Mutex` ，那就不好搞啦，因为 `spawn` 创建的子线程如若要调用 `Mutex` ，则 `Mutex` 所有权进入子线程，后面创建的子线程就不能用了，如下：

```rust
use std::sync::Mutex;
use std::thread;
fn main() {
    let counter = Mutex::new(0);
    let mut handles = vec![];
    for _ in 0..10 {
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
// 这段代码无法编译通过
```

但是我们也不能用之前提到过的智能指针 `Rc<T>` 实现多引用，我们需要一个和它类似的类型。

从例子也可以看出， `Mutex` 也是具有内部可变性的。

#### `Arc<T>` 原子引用计数

使用例：

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

