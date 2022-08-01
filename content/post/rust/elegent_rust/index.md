---
title: 优雅的Rust
date: 2020-09-15T15:30:44+08:00
slug: elegent_rust
image: cover.png
categories:
    - Rust
tags:
    - Basic
    - Rust
---

小记自己为什么会学 `Rust` . `Rust` 有什么特性吸引到我了~

---

如果说 `C` 语言是具备高灵活性, 信任着开发者, 将一切交给开发者来解决的话.

那么 `Rust` 就是降低该灵活性, 毫不信任开发者, 将开发者的大部分错误都在编译期发掘出来, 强制要求开发者去了解可能存在的问题并解决掉它.

 `C` 语言可能条条大路通罗马,  `Rust` 可能只有寥寥无几的道路. 

总之, `Rust` 拔高了开发者的下限, 如果水平不够是连编译器都不会通过的, 更不要说 `commit` 了.

# 优雅的Rust

听说到 `Rust` 最多的自然是所谓的内存安全. 那么什么是内存安全？ 内存安全体现在哪里呢？

比如说这段 `C语言` 代码.

```c
int a[5];
a[6] = 6;
printf("%d", a[6]);
```

这段代码能正确输出 `a[6]` 吗?

答案是可以的, 虽然它很明显的内存越界了, 但内存越界似乎在 `C` 里并不是什么`大不了`的事, 编写的代码随便越界, 只不过导致的后果需要自己承担而已.

当然现今大部分的高级语言对指针越界都会有很明确的编译错误, 所以仅这点来看并不能体现出 `Rust` 的特点, 只不过是有了现代语言该有的东西——内存越界提示.

那么有什么可以体现出 `Rust` 独有特性的地方呢?

参考这一段结构体.

```c
typedef struct {
    struct Node *next;
    int val;
}Node;

typedef struct {
    Node *head;
    Node *tail;
    int size;
}LinkedList;

struct {
    void (*push)(struct LinkedList *self, Node val);
    Node *(*peek)(struct LinkedList *self);
    Node *(*pop)(struct LinkedList *self);
}StackMethod;
```

这是一个简单的数据结构栈, 在栈方法中定义了三个函数指针, 不需要关注 `push` 方法, 来看看 `peek` 方法.

`peek` 方法将栈中的尾结点返回, 和 `pop` 方法的区别是它不会从链表中删除该节点. 但此时该方法的调用者拥有了该节点的所有权限, 如果调用者此时又释放了该地址指向的内存, 当再次调用 `pop` 方法时, `pop` 方法将返回一个已经释放了内存的结点指针, 此时如果访问该结点大概率会出现 `Segementation fault`. 因为访问了一个没有分配的内存空间. `Segementation fault` 大概是我写 `C` 时候最讨厌的错误了.

那么 `Rust` 是如何解决的呢? 答案是`所有权`.

## 所有权

```rust
fn main() {
    let s1 = String::from("hello");
    let s2 = s1;
}
```

在将 `s1` 赋值给 `s2` 后, 此时 `s1` 就已经是无效的了, 它的作用域仅到赋值给 `s2` 之前为止.

同样类似的.

```rust
fn hello(str: String) {
    println!("{}", str);
}

fn main() {
    let s1 = "hello".to_string();
    hello(s1);
    println!("{}", s1);  // error
}
```

这段相等同的 `C` 代码会有问题么?

可能会有问题. 如果在 `hello` 中释放了内存, 那么 `main` 函数中后续对 `s1` 的访问就会出现段错误. 就像这样:

```c
void hello(char* str) {
    printf("%s\n", str);
    free(str);
}
int main() {
    char s1[5] = { 'h', 'e', 'l', 'l', 'o'};
    hello(s1);
    printf("%s\n", s1);
}
```

在 `Rust` 中就完全避免了这种情况, 这样的代码在Rust中连编译都不会通过的.

那么 `main` 函数后续不能再用该变量了吗? 有两种方法可以解决该问题.

可以将 `hello` 函数的签名增加个返回值, 使用完变量后将其所有权返回给调用者. `main` 函数也需要提供一个接收者接收该变量的所有, 该做法使用了 `Rust` 变量覆盖的特性, 可以声明同名变量, 就跟 `Javascript` 类似. 后续声明的同名变量会覆盖先前的.

```rust
fn hello(str: String) ->String {
    println("{}", str);
    str
}

fn main() {
    let s1 = "hello".to_string();
    let s1 = hello(s1);
    println!("{}", s1);
}
```

第一种是个很自然而且很容易想到并理解的做法( `Rust` 中函数块最后一行且没有带分号的会被当做返回值, 所以可以省略掉 `return` ). 还有另一种做法, 那就是引用.

```rust
fn hello(str: &String) {
    println!("{}", *str);
}

fn main() {
    let s1 = "hello".to_string();
    hello(&s1);
    println!("{}", s1);
}
```

使用  `&` 表示一个引用或引用类型, `*` 则表现为消除 `&`, 也就是解引用. 用法类似与 `C` 中的指针. 而在 `Rust` 中是会自动解引用的, 所以 `hello` 函数也可以改为:

```rust
fn hello(str: &String){
    println!("{}", str);
}
```

在 `C` 中可能存在的野指针问题在 `Rust` 中也可以通过所有权系统来解决.

> 野指针主要为未初始化或释放后未置空或操作超越变量作用域.

参照如下 `C` 代码:

```c
#include <stdio.h>
#include <stdlib.h>
int main(void) {
    int *p = (int *)malloc(4);
    *p = 4;
    free(p);
    int *q = (int *)malloc(4);
    *q = 2;
    printf("q = %d\n", *q);  //此处打印 q = 2
    *p = 4;
    printf("q = %d\n", *q);  //此处打印 q = 4
}
```

是不是很神奇. 那么为什么会出现这种现象?

因为在 `C` 中使用 `malloc` 分配内存的时候, 会优先分配前次释放的内存, 所以两处调用 `malloc` 所得到的地址是一样的. 在释放 `p` 之后, 没有对 `p` 进行赋空操作, 此时 `p` 就成了一个野指针. 然后接着又分配了一个跟前序 `p` 一样大小的内存空间, 此时 `p` 和 `q` 就是一样的了——指向相同的地址. 所以对 `p` 指向的值进行的修改会反映在 `q` 上. 虽然简单的在释放 `p` 后给其赋值为 `NULL` 就可以了, 但是如果有个人疏忽了呢? 毕竟人是不会永远不犯错的. 在 `Rust` 中类似这样的代码是不可能通过的, 因为在释放 `p` 的时候所有权已经转移给 `free` 函数了,  `main`函数后续是不能再使用的.

不管是学过 `Java` 的还是学过 `C` 的, 肯定都多多少少了解过关于引用传递和值传递, 有相当多的教学声明 `Java` 中仅存在值传递. 

在 `C` 语言中指针则类似于引用传递, 通过地址引用指向内存空间的值. 因为有地址所以可以修改其中的值并且被其它函数观察到. 传递引用有一个问题, 有时候我们并不想别人改变我们的值, 但又不得不声明为指针(比如说结构体的自引用), 同时又忘了给该参数声明为 `const` , 这样就会有很多问题. 尤其在初学 `C` 的时候, 几乎很少有人会用到 `const` 关键词. 很多人是不晓得该关键词到底有多么重要的, 而 `Rust` 就会强迫我们去学习, 了解不可变的重要性.

那么很自然的就过渡到下一个阶段~

## 可变性与不可变性

然而在 `Rust` 中所有的变量如果不显式声明, 那就是不可修改的( `unsafe` 是特例). 所以如果要传递一个可变引用, 就需要如下的声明:

```rust
fn hello(str: &mut String) {
    println!("{}", str);
    str.push_str("world"); //3. 修改str的值.
}

fn main() {
    let mut s1 = "hello".to_string(); //1. 此处声明s1是可变的变量.
    hello(&mut s1);  //2. 传递一个可变引用给hello函数.
    println!("{}", s1);
}
```

显式声明 `str` 是可变的, 否则对变量做的任何修改在编译期都是无法通过的. 这就避免了很多 `Code` 时的隐性错误.

`Rust` 中关于该特性有几个关键点.

1. 变量可以有多个不可变引用.
2. 变量只能有一个可变引用.
3. 变量有可变引用的同时不能有不可变引用.

比如如下代码:

```rust
fn main() {
    let mut s1 = "hello".to_string();
    let s2 = &s1;
    hello(&mut s1);
    println!("{}", s1);
    println!("{}", s2);
}
```

编译时会报错

> error[E0502]: cannot borrow `s1` as mutable because it is also borrowed as immutable

错误提示相当友好了, 不能将 `s1` 作为可变引用因为它已经有一个不可变引用了. 为什么要这么设计?

可以想象一下在 `C` 中, 编写了一个返回指针的函数, 有多个不同调用方或许都对指向的值进行了修改, 某个调用方又如何知道该值是否是最初的函数所返回的呢? 还是因为各个调用方可能进行的修改后的结果. 解决方法肯定是有的, 比如说注意一下各个调用方的顺序之类的, 只是有时项目庞大而复杂, 是没法顾及到方方面面, 这时是需要有个 "人" 来提醒一下我们.

因此, 可变引用与不可变引用直接是互斥的, 不可变引用之间也是互斥的, 它们不能同时存在.

上面的代码调换一下位置就可以了.

```rust
fn main() {
    let mut s1 = "hello".to_string();
    hello(&mut s1);
    let s2 = &s1;
    println!("{}", s1);
    println!("{}", s2);
}
```

此时在 `s2` 声明的位置, `s1` 的可变引用的作用域已经结束, 现在 `s1` 是没有其它引用的, 所以可以拿到其的一个引用.

---

在这里, 为了引入个下一话题, 我们不自然地回顾一下上一章节的野指针问题~

类似的如下 `C` 代码:

```c
#include <stdio.h>
char *dangle() {
    char a[3] = "abc";
    printf("%s\n", a);
    return a;
}
int main(void) {
    char *p = dangle();
    printf("%s\n", p);
}
```

在函数结束的时候变量 `a` 已经被销毁, 将该地址返回就会产生野指针.

如何修正该代码?

```c
#include <stdio.h>
char *dangle() {
    char *a = "abc";
    printf("%s\n", a);
    return a;
}
int main(void) {
    char *p = dangle();
    printf("%s\n", p);
}
```

这样就可以了.这样所声明的变量 `a` 实际上为字符串常量, 是不可变的, 存储在静态存储区. 由程序结束后操作系统回收该部分内存. 所以在函数结束后 `a` 指向的值是不会被销毁的. 将该地址返回也是完全没有错误的.

这些细枝末节在初学 `C` 的时候是很难注意到的. 毕竟大多数大学生使用的教材还是谭浩强版, 用于应试是够了, 实际开发是远远不行的.

如果在 `Rust` 中写出这样的代码呢?

```rust
fn main() {
    let reference_to_nothing = dangle();
}

fn dangle() -> &String {
    let s = String::from("hello");
    &s
}
```

首先说点题外话, 在 `Rust` 中命名通常用蛇形命名法, 不像 `Java` 中普遍的是大驼峰和小驼峰. 至于好看与否, 看眼缘吧~

这里同样的编译是不能通过的.

> error[E0106]: missing lifetime specifier
> --> src\main.rs:5:16
> |
> 5 | fn dangle() -> &String {
> |                ^ expected named lifetime parameter
> |
> = help: this function's return type contains a borrowed value, but there is no value for it to be borrowed from
> help: consider using the `'static` lifetime
> |
> 5 | fn dangle() -> &'static String {
> |                ^^^^^^^^

可以说相当友好了, 编译器同时提示了错误是什么, 以及该如何解决它.

该错误说明了该函数缺少生命周期指示符. 帮助说明了该函数返回的值包含了一个引用值, 但是这种类型是没有可以用来借用的, 需要考虑加上 `'static` 生命周期, 这样就杜绝了返回局部变量引用的情况.

这样就引入了下一个话题~

## 生命周期

以下内容主要参考 [The Rust Programming Language](https://doc.rust-lang.org/book/ch10-03-lifetime-syntax.html)

生命周期只是一个标签, 语法是`'`+字符串/字符. 比如 `'a`, `'b`. `'static` 是一个特殊的生命周期, 表示静态存储, 类似于 `Java` 和 `C\C++` 中的 `static`, 也就是一直存活到程序结束运行才会销毁.

生命周期主要是为了避免悬垂引用(野指针), 在 `Rust编译器` 中实现了一个借用检查器, 通过比较作用域来确保所有借用的有效性.

```rust
{
    let r;                // ---------+-- 'a
    {                     //          |
        let x = 5;        // -+-- 'b  |
        r = &x;           //  |       |
    }                     // -+       |
    println!("r: {}", r); //          |
}                         // ---------+  
```

这里 `r` 的生命周期为 `'a`, `x` 的生命周期为 `'b` ,  `'b` 的范围小于 `'a` 的范围, 由于 `r` 引用了一个存活范围小于它的变量, 所以会编译失败.

```rust
{
    let x = 5;            // ----------+-- 'b
    let r = &x;           // --+-- 'a  |
    println!("r: {}", r); //   |       |
                          // --+       |
}                         // ----------+
```

改为这样子, 因为 `r` 的生命周期 `'a` 小于 `'b` , 这样就能保证在 `r` 有效的时候 `x` 总是有效的, 这样 `r` 就不可能变成一个野指针.

考虑这样一个函数:

```rust
fn longest(x: &str, y: &str) -> &str {
    if x.len() > y.len() {
        x
    } else {
        y
    }
}

fn main() {
    let string1 = String::from("long string is long");
    {
        let string2 = String::from("xyz");
        let result = longest(string1.as_str(), string2.as_str());
        println!("The longest string is {}", result);
    }
}
```

乍看之下没有什么问题对吧? 但是编译还是不通过.

>error[E0106]: missing lifetime specifier
>--> src\main.rs:1:33
>|
>1 | fn longest(x: &str, y: &str) -> &str {
>|               ----     ----     ^ expected named lifetime parameter
>|
>= help: this function's return type contains a borrowed value, but the signature does not say whether it is borrowed from `x` or `y`
>help: consider introducing a named lifetime parameter
>|
>1 | fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
>|           ^^^^    ^^^^^^^     ^^^^^^^     ^^^

根据报错提示, 我们修改函数签名为`fn longest<'a>(x: &'a str, y: &'a str) -> &'a str` 这样这段代码就可以正确运行了. 生命周期在函数签名中的用法类似于泛型, 前面多个单引号而已. 然后在对应的参数上声明所需要的生命周期就可以了.

这个函数签名表示告诉调用者 `x` , `y` , `str` 这三个至少能存活的一样久, 否则就不能编译通过.

我们修改下 `main` 函数为如下:

```rust
fn main() {
    let string1 = String::from("long string is long");
    let result;
    {
        let string2 = String::from("xyz");
        result = longest(string1.as_str(), string2.as_str());
    }
    println!("The longest string is {}", result);
}
```

我们将延长 `result` 的生命周期, 并且将 `println` 移动到块外面, 这样子编译是不通过的. 来看看编译错误信息:

>error[E0597]: `string2` does not live long enough
>--> src\main.rs:14:44
>|
>14 |         result = longest(string1.as_str(), string2.as_str());
>|                                            ^^^^^^^ borrowed value does not live long enough
>15 |     }
>|     - `string2` dropped here while still borrowed
>16 |     println!("The longest string is {}", result);
>|                                          ------ borrow later used here

提示我们 `string2` 在 `println` 前就被释放了. 如果像之前那样没有给函数签名加上生命周期的话, 编译器是不能知道函数返回的值到底能存活多久的, 生命周期表明了返回的值与参数的生命周期应该是一致的, 这里所谓的一致是指同样生命周期标签中最小的那个. 在正确的代码中, 三个参数分别对应的 `string1`, `string2`, `result` 中, 最短的那个是 `result` , 编译器能推断出在 `result` 存活的范围它所引用的值是一直有效的.

而在错误的代码中, `string2` 是生命周期最短的那个, 所以编译器推断出 `result` 只能在 `string2` 存活的范围内使用, 一旦超出该范围, 就可以提示调用者需要修改代码.

简单说下 `Rust` 如何清理内存, 只要让它离开作用域就可以了, 离开作用域的时候会自动调用该类型的 `drop` 方法, 而 `drop` 一般都是由编译器提供默认实现的, 有时候默认实现不够高效才需要自己去实现. 我们不能调用变量的 `drop` 方法, 那样就会有多次释放的问题, 如有需要可以调用标准库提供的 `drop` 函数.

标准库有提供一个 `drop` 函数, 用来手动释放内存, 可以看看它的实现~

> core::mem
> pub fn drop<T>(_x: T)
> Disposes of a value.
> This does so by calling the argument's implementation of Drop.
> This effectively does nothing for types which implement Copy, e.g. integers. Such values are copied and then moved into the function, so the value persists after this function call.
> This function is not magic; it is literally defined as
> pub fn drop<T>(_x: T) { }
>
> Because _x is moved into the function, it is automatically dropped before the function returns.

 它的实现仅仅是一个空函数块, 将该变量的所有权转移到该函数块中, 在结束的时候调用该变量的 `drop` 方法.

---

上面似乎都是一些内存安全相关的特性, 感觉只体现出来 `Rust` 编译器的强大, 接下来, 可以转下一个话题了, 简单介绍下 `Rust` 的一些优雅的语法特性~

## Trait

`Trait` 类似与 `Java` 中的接口概念, 近年来一直以来都有组合大于继承的口号, `Trait` 就是在 `Rust` 中用来实现组合这一理念的.

继承有时候用起来是真的不顺心. 继承吧, 好多用不到的东西, 不继承吧, 又有很多重复工作要做. `Rust` 中干脆扔掉了继承.

像前面讲到的实现 `drop` 方法, 怎么为我们的类型实现 `drop` 方法呢?

`Drop` 中只有一个方法, 它的签名为: `fn drop(&mut self);` 所以我们就像实现接口一样实现该方法就可以了.

```rust
struct MyVar {
    names: Vec<String>
}
impl Drop for MyVar {
    fn drop(&mut self) {
        &self.names;
        println!("MyVar drops");
    }
}
fn main() {
    let _ = MyVar {
        names: vec!["hello".to_string(), "world".to_string()]
    };
}
```

运行后将打印出 `MyVar drops` , 可以看出来在离开作用域后就自动调用了我们自定义的 `drop` 方法.

可以用 `Trait` 配合泛型编写一个通用的 `largest` 函数.

```rust
fn largest<T: PartialOrd + Clone>(list: &[T]) -> T {
    let mut largest = list[0].clone();
    for item in list.iter() {
        if *item > largest {
            largest = item.clone();
        }
    }
    largest
}
fn main() {
    let number_list = vec![34, 50, 25, 100, 65];
    let result = largest(&number_list);
    println!("The largest number is {}", result);
    let char_list = vec!['y', 'm', 'a', 'q'];
    let result = largest(&char_list);
    println!("The largest char is {}", result);
}
```

`PartialOrd` 是基于排序为目的而比较一个类型, 实现了该 `Trait` 的类型就可以使用 `>` ,  `<=` , `>` , `>=` , 可以近似的看作 `C++` 中的操作符重载.

这里再简单说明一下 `Copy` 和 `Clone`.

`Clone` 就是所谓的 `深拷贝` , 我们也可以用 `Copy` , `Copy` 是浅拷贝.

`Copy` 是简单的拷贝存储再栈上的位来赋值值, 比较高效, 是一个标记型 `Trait` , 该 `Trait` 的实现基准是: 如果一个类型内部的类型全部是 `Copy` 的, 那么该类型也是 `Copy` , 比如所有的基本类型. 而 `Clone` 是深拷贝, 我们需要为必要的类型自己实现该 `Trait` , 当调用一个 `Clone` 的时候我们应该知道该操作可能会比较慢.

我们也可以为 `Triat` 实现 `Trait` , 这用到了 `Rust` 中的动态分发, 这里不详细说明, 只做介绍.

```rust
trait Animal {
    fn name(&self) -> String;
}
trait Bark {
    fn bark(&self);
}
struct Dog;
impl Animal for Dog {
    fn name(&self) -> String {
        "Kitty".to_string()
    }
}
impl Bark for Box<dyn Animal> {
    fn bark(&self) {
        println!("{} bark.", self.name());
    }
}
fn main() {
    let dog: Box<dyn Animal> = Box::new(Dog);
    dog.bark();
}
```

我们可以用 `Trait` 来组合成我们最终所需要的类型, 每个 `Trait` 都是那么的小巧简洁~ 不像是继承, 继承有着严重的耦合, 并且显得不是那么精巧.

---

之前我们有提到过 `Rust` 中是没有空指针的, 那么我们如何表示一个可能存在的值呢?

如果有人用过 `JDK8` 中的 `Optional` 的话, 那应该就很熟悉了~答案就是 `Option`.

`Optional` 在 `Java` 中出现的太晚了, 很多人对这个特性是不怎么了解的, 迁移起来也很麻烦, 普遍还是使用空指针和捕获空指针异常来处理.

而在 `Rust` 中 `Option` 遍布了各个角落, `Option` 只是一个简单的枚举, 所以我们直接说明下 `Rust` 中使用枚举处理空指针和异常错误的方法.

## 枚举

### Option

`Option` 只是一个简单的枚举类型, 但是在 `Rust` 中枚举是很强大的.

`Option` 的签名是

```rust
pub enum Option<T> {
    None,
    Some(T),
}
```

它是这么使用的:

```rust
fn main() {
    let x = Option::Some(1);
    match x {
        Some(x) => println!("Val is: {}", x),
        None => println!("No val")
    };
}
```

在 `Rust` 中对于枚举的所有类型都强制遍历处理, 当然也可以用 `_` 来忽略掉不需要的部分, 所有我们不确定是否有值的地方都需要用 `Option` 来表示.

### Result

`Rust` 中也是用枚举来处理异常错误.

```rust
use std::net::TcpStream;
fn main() {
    let tcp_stream = TcpStream::connect("127.0.0.1:8000");
    match tcp_stream {
        Ok(_) => println!("success"),
        Err(e) => println!("{}", e),
    };
}
```

不需要捕获异常, 要知道 `Java` 中捕获异常是很耗费性能的, 使用枚举来处理空指针和异常错误是不是很优雅~

除了使用模式匹配外, 我们还可以使用 `if let` , 如下:

```rust
use std::net::TcpStream;
fn main() {
    let tcp_stream = TcpStream::connect("127.0.0.1:8000");
    if let Ok(_) = tcp_stream {
        println!("success");
    } else {
        println!("error");
    }
}
```

同样类似的还有 `while let` .

## Summary

上述差不多说明了一些为什么我选择学 `Rust` 的理由, 当然还有很多其它的, 我所述并不足以解释 `Rust` 在最受程序员欢迎的语言调查中占据榜一.

我在写 `C` 代码的时候, 总会纠结该不该释放内存, 有没有野指针的可能. 也烦恼与标准库不够好用, 各平台头文件不够统一, 没法编写多平台通用的代码.

同样的, 在写 `Java` 的时候又会感觉有些啰嗦, 很多写法不够简洁.

而 `Rust` 的话, 标准库足够好用, 是否内存安全有编译器提醒, 而且写起来有很多语法糖比 `Java` 要小巧很多, 也只有很小的运行时, 没有垃圾回收, 可以直接打包成二进制文件运行. 性能与 `C` 是差不多的.

不过说来我学 `Rust` 还有一个原因是因为它够难, 而且没什么历史包袱, 比较有挑战. `C++` 的历史包袱换个词也可以说是经验积累, 不过我个人觉得有些太过厚重, 有些难啃吧.

本文浅尝辄止, 简单说明了一些 `Rust` 的特性. 这些特性差不多都是众多程序员在编写 `C/C++` 程序中总结出来的类似于最佳实践之类的东西. 所以本文也大量对比了 `Rust` 和 `C` , 并且学习 `Rust` 也是离不开 `C` 的, 其中存在了 `unsafe` 这样的东西. 不了解 `C` 的话是没法写 `unsafe` 部分的, 当然这部分大部分程序员也不怎么会接触到.

同时 `Rust` 作为系统级语言, 与 `C` 也有着非常友好的交互性, 很容易使用的 `FFI` , 不管是为了从 `C` 逐步切换到 `Rust` , 还是为了牺牲点安全提高灵活性的目的而使用 `FFI` , 这都算是一个亮点了. 毕竟 `C` 在各个领域都算是不可或缺的基本编程语言了.


[^野指针]: 野指针主要为未初始化或释放后未置空或操作超越变量作用域.

