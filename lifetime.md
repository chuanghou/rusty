# rust lifetime

rust 生命周期是个比较新的概念，我也纳闷，我竟然看了三遍才完全看明白，前两次看，每次总是看的似懂非懂，最近因为公司项目告一段落，可以开始花点时间看，才又开始看这个东西。

### Ref的本质
太阳底下并没有什么新鲜事，其实本质就是C/C++的指针，这里我并不和C++中的引用相比较，因为C++中的引用也就是指针套壳罢了。只是对于rust来说，所有权的概念，使得引用相对特殊，引用可以使用对象，但是随着引用本身的析构，并不会照成对象的析构，其实这和C++里面也类似，栈上的指针变量析构，对象本身并不会析构，但是rust引用的特殊之处在于，引用本身其实和原始拥有内存权限的那个变量在编译过程中维护了一个相互关系，对于编译器来说，持有内存所有权的对象一旦析构，对应的引用肯定不能用了，如果你还在用，我指定编译不通过。

### 函数的返回值问题
上文提到了因为引用的和主变量的依赖关系，可以在编译的时候进行相应分析保证了内存的安全性，但是如果一个引用是一个函数返回的，这就有点麻烦了，因为你并不知道这个引用什么时候不能用了，这看似是个无解的问题。类比而言，如果在C++中你使用了一个函数返回的指针，你并不知道，这个指针指向的对象什么时候析构掉，完全就不能用。但是rust的设计者们解决了这个问题。
### 函数返回的引用问题
#### 1. 无参函数
>最为特殊的情况，假设一个函数返回一个引用，函数无参，这种情况其实反而简单了，因为此时函数返回的引用肯定是静态生命周期的，也就是整个程序运行过程中都可以用的引用，也许你会疑惑，我在函数内部申请一块内存，返回引用不行吗？实际是真不行，直接就编译失败，其实这里问题很清楚了，因为，因为s对象持有所有权，如果函数结束，s直接就析构了，返回个引用回去，直接就完蛋
```rust
fn empty_parameters_with_return_ref() -> &String{
    let s = String::from("Hello");
    &s
}

   | fn empty_parameters_with_return_ref() -> &String{
   |                                          ^ expected named lifetime parameter
   |
   = help: this function's return type contains a borrowed value, but there is no value for it to be borrowed from

```

>可能有些人注意到了，rust的编译提示，需要lifetime参数，觉得我说得不对，ok，我们加一下,你看问题很清楚了吧，returns a reference to data owned by the current function, 
```rust
fn empty_parameters_with_return_ref<'a>() -> &'a String {
    let s = String::from("Hello");
    &s
}
error[E0515]: cannot return reference to local variable `s`
  --> src/main.rs:19:5
   |
   |     &s
   |     ^^ returns a reference to data owned by the current function
```
> 还有如下这种静态周期情况，这种是ok的，因为内部返回了一个全局周期的引用量
```rust
fn empty_parameters_with_return_ref<'a>() -> &'a str {
  "aaa"
}
```

#### 2.有参情况
其实这才是这个问题的比较核心的问题，上文所描述的无参情况其实一句话概括为，函数内部申请的内存无法以引用的形式返回，只能返回一些静态全局周期的量。但是如果本身函数的参数有引用变量，而返回的引用变量很有可能从和参数相关，那么就不能一锤子打死，不用参数对应的引用周期了。rust因为这个事总结了三种规则，在我看来这三个规则，真的是容易让人误入歧途，因为他提供的并不是一种逻辑的思维过程，而用一种类似盲人摸象的经验注意告诉开发者，说心里话这件事是有点让我失望的，感觉作者团队的智力水平远远到不了卓越水平，可能吧，搞程序确实不用太卓越，
> 从我的角度上理解有参函数的声明周期，其实他是一种内外部的协议，假设我给参数和返回值标注了标注了生命周期，那么函数内部和外部就可以使用这种规定完成内外部的编译，对于调用者来说，你既然给与了我说明这几个参数和这几个返回值之间的引用失效关系，ok，我只要外部的使用符合了这个生命周期参数的限制，其他我不管了，对于函数内部，我们既然提供了这种生命周期保证，我们内部反正是能拿到所有的对应关系（参数与返回值），那么我们就分析一下，这种保证能不能实现呢，其实这就是使用生命周期的原理，
>
## 书中规则的解释

>The first rule is that the compiler assigns a lifetime parameter to each parameter that’s a reference. In other words, a function with one parameter gets one lifetime parameter: fn foo<'a>(x: &'a i32) ; a function with two parameters gets two separate lifetime parameters: fn foo<'a, 'b>(x: &'a i32, y: &'b i32) ; and so on.

>The second rule is that, if there is exactly one input lifetime parameter, that lifetime is assigned to all output lifetime parameters: fn foo<'a>(x: &'a i32) -> &'a i32. 

The third rule is that, if there are multiple input lifetime parameters, but one of them is &self or &mut self because this is a method, the lifetime of self is assigned to all output lifetime parameters. This third rule makes methods much nicer to read and write because fewer symbols are necessary.

以上为三个rule，其实这三个不能算rule，他只是在代码作者没填生命周期的情况下，默认给函数或者impl加入的生命周期参数，而且搞笑的是，你即使函数满足上述条件，编译器在编译的时候自动给加上了周期，也不代表就一定正确，也会编译失败，所以这三个规则只能算是一种不太完备的语法糖

首先说第一个规则，每参数都会获得一个不同于其他参数的生命周期参数，继续，如果入口参数只有一个，那么所有的出参也都将获得这个参数，相当于把入口和出口绑定，这俩必须共死，有一个死了，另外一个必须死，因为我们前面说了，返回的参数如果不是参数的引用只能来至于内部的静态全局引用，所以其实绑死问题不大，唯一的问题是，返回值是内部的静态全局引用，通过这种方式，限制了返回值最长使用区间只能和入口参数一致，其实他可以更长了，代码如下，其实你看，这里返回的是静态变量，但是因为默认添加生命周期的原因，导致编译不通过，因为此时默认添加的编译参数导致了，入口参数和出口返回值，生命周期锁到一起了，参数到期了，然后返回值也不能用了，其实本身返回值是可以用的
```rust
fn transfer(p : &String) -> &str {
    "sss"
}

fn main() {

    let ss: &str;
    {
        let s = &String::from("Hello, world!");
        ss = transfer(s);
    }
    println!("{}", ss);
    
}

error[E0716]: temporary value dropped while borrowed
  --> src/main.rs:27:18
   |
27 |         let s = &String::from("Hello, world!");
   |                  ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ creates a temporary value which is freed while still in use
28 |         ss = transfer(s);
29 |     }
   |     - temporary value is freed at the end of this statement
30 |     println!("{}", ss);
   |                    -- borrow later used here
   |
   = note: consider using a `let` binding to create a longer lived value
```

如果稍微改动
```rust
fn transfer<'a, 'b>(p : &'b String) -> &'a str {
    "sss"
}

fn main() {
    let ss: &str;
    {
        let s = &String::from("Hello, world!");
        ss = transfer(s);
    }
    println!("{}", ss);
}
```
或者简单一点，这两种情况下，都是ok的，因为'a只是函数定了一个泛型参数，编译的时候，发现其实a应该是静态生命周期，而且只和静态生命相关，那就ok了
```rust
fn transfer<'a>(p : & String) -> &'a str {
    "sss"
}

fn main() {
    let ss: &str;
    {
        let s = &String::from("Hello, world!");
        ss = transfer(s);
    }
    println!("{}", ss);
}
```
以上只是特殊情况，一般情况下，通过这种默认的生命周期标注方法，是可以解决大部分问题的

>  第三条规则一开始是最为困惑我的，到最后才发现其实本质是语言理解问题，带有self的引用本身和普通引用没有区别，只是使用这种标注方式，大概率是可以通过编译的，所以编译器对于没有标注引用的情况，采用了这种标注方式，但是其实这种标注方案，是会编译不通过的。如下

```rust
struct MyRef {
    s: String,
}
impl MyRef {
    fn print(&self, s : &String) -> &String {
        s
    }
}

fn main() {
    let ss: &String = &String::from("sss");
    let ret: &String;
    {
        let my_ref = MyRef{s: String::from("my_ref")};
        ret = my_ref.print(ss);
    }
    println!("{}", ret);
}

  --> src/main.rs:11:9
   |
10 |     fn print(&self, s : &String) -> &String {
   |              -          - let's call the lifetime of this reference `'1`
   |              |
   |              let's call the lifetime of this reference `'2`
11 |         s
   |         ^ method was supposed to return data with lifetime `'2` but it is returning data with lifetime `'1`
   |
help: consider introducing a named lifetime parameter and update trait if needed
   |
10 |     fn print<'a>(&self, s : &'a String) -> &'a String {
   |             ++++             ++             ++
```

## 总结

生命周期本质是一种约束，凡是标注了相同生命周期参数的变量，只有有一个析构了，另外一个就不能用了，虽然我们知道这个对象的内存还没析构，但是因为生命周期的约束，不能用了，编译会不通过，针对没有添加生命周期的函数，impl，编译器会根据一定的规则加一些默认分配，但是这些分配并不能一定保证可以通过编译，最终生命周期的编译过程是编译器保证的，编译器并不能代码作者选择出合理的生命周期配置，但是可以校验你选了这种生命周期配置之后，内存有没有问题，他自己会试着给出基本的搭配。