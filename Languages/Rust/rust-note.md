# 内存安全

Rust的作为一个内存安全的编程语言，避免以下内存安全问题发生：

- 未初始化指针；
- 空指针；
- 缓存区溢出；
- 悬垂指针；
- 二次释放指针或释放未分配内存的指针。

通过以下几方面语言层面的设计，rust避免上述问题的出现。

## 变量初始化

Rust禁止使用未初始化的变量，在编译阶段对变量是否初始化进行检查，若存在未初始化的变量，编译将无法通过，这样避免了使用**未初始化指针**。

## `Option<T>`枚举类型

Rust中无空指针概念，使用`Option::None`代表*无值*。当需检查一个变量是否有值时，可通过`match`表达式对`Option<T>`类型变量进行检测，匹配`Some<T>`则代表值存在，否则匹配`None`表示无值。而`Option<T>`类型的值必须进行模式匹配才能获取，这样避免了直接使用无值的变量。`Option<T>`的设计也使Rust避免了类似使用**空指针**所产生的内存安全问题。

## 数组边界检查

Rust**索引数组**时，对数组边界进行检查，避免越界；对于切片，若索引值超过底层数组长度，将重新分配空间，并把原底层数组数据复制到新的内存中。上述规则避免了**缓冲区溢出**。

## 生命周期系统

**生命周期系统**保证在编译期就能检查并消除**悬垂指针**。

## 所有权系统

**所有权系统**使内存的**分配与释放**按照所有权规则进行，避免手动分配和释放内存导致的二次释放或释放未分配的内存等问题。

# 所有权

## 变量与值的基本关系

在Rust中，变量称为值**所有者**，值有且只有一个所有者，所有者具有值的**所有权**，当变量离开作用域时值被释放掉。从底层角度看，在**堆**中分配内存返回一个指向该内存的**指针**，在Rust中此**指针**为内存所有者，指针存储在**栈**中，指针离开栈时内存被释放掉。

变量与值是一种绑定关系，变量通过`let`声明，在初始化式或赋值表达式中，变量与值进行绑定，变量成为值的所有者;

```rust
let a = Box::new(5);  // Box::new()在堆中分配内存，用于储存5，a为5的所有者
```

当变量离开作用域时，值被释放。

```rust
{
    let a = Box::new(5);
} // a离开作用域后，值被释放掉
```

## 所有权移动与借用

### 移动

在赋值表达式中，**左值**获取**右值**的所有权；另一方面非*字面值*的右值将失去所有权，这种失去所有权的过程为**所有权移动**。所有权移动重要功能是，保证无法通过失去所有权的变量访问值。

```rust
let a = Box::new(4);
let b = a; // b获取a所有权，a失去所有权
println!("{}", a); // err，此时a无法访问
```

### 借用

若仅仅为了使用变量的值，而不是获取变量的所有权，可使用`&`操作符引用*值*，此过程称为**所有权借用**；

```rust
let a = Box::new(5);
let b = &a; // 借用a的所有权
println!("{}", b); // 通过b访问了a的值，结果为6
```

借用所用权的变量离开作用域不会导致值的释放。

```rust
let a = Box::new(6);
{
    let b = &a;
} // b离开作用域后，a的值不会释放
println!("{}", a); // 结果为6
```

`&`操作符号获取变量的**引用**，通过**解引用**操作符*****获取值。

```rust
let a = Box::new(7);
let b = &a;
println!("{}", b); // 对b自动解引用，结果为7
let c = *b; // err，可以通过*b访问a的值，*b视为a的别名，但是*b无所有权，无法移动所有权。
```

在rust中，自动解引用发生在大部分表达式中，这样屏蔽了手动解引用，使大部分表达式可以直接操作类似`&a`**引用**，由于`&`操作不获取或`Move`或`Copy`所有权，所以`&`操作成为名副其实的**借用**操作。

## 表达式与所有权

### 表达式

Rust语言的基本单元是表达式，分为**位置表达式**和**值表达式**，位置表达式包括：局部变量、静态变量、解引用表达式、数组索引表达式、字段引用表达式；其它所有表达式都是值表达式。

#### 操作符

表达式与表达式之间通过操作符相联，构成新的表达式。与其它编程语言类似rust也有许多操作符，例如`+`、`==`等操作符，在rust中每一个值都归属于一种**类型**，而每一种**类型**实际上是**一组操作的集合**，或者说类型实现了一组**trait**。比如，`i32`类型可以进行`+`、`-`、`*`、`/`等运算，实际上是因为`i32`类型*实现*了一组trait`Add<i32>`、`Mul<i32>`等。

```rust
impl Add<i32> for i32 {
    type Output = i32;
    fn add(self, other: Output) -> Output {
        self + other
    }
} // i32类型实现了Add<i32>
let a = 23.add(4);
let b = 23 + 4; // 等同于23.add(4)
```

即，rust中的操作符是**方法调用**的*语法糖*。

#### 表达式上下文

Rust中，仍有一部分操作符的功能不由trait定义，包括：

- 函数调用操作符`()`
- 赋值操作符`=`
- 借用操作符`&`
- 解引用操作符`*`（用于操作指针）
- 字段引用操作符`.`

另，索引表达式实际上是实现了`std::ops::{Index,IndexMut};`，`v[index] = value`是表达式`*v.index(index)`的语法糖，`v`实际上是处于借用表达式上下文中，所以这里不再赘述索引表达式的所有权问题。

那么**表达式上下文**可以划分为：

- 赋值表达式操作数
- 函数调用传入参数
- 一元**借用**操作数
- **解引用**操作数
- 字段表达式的操作数
- `let`语句的初始化式
- `if let`、`while let`和`match`表达式中的检查表达式
- 闭包

### 赋值表达式及初始化式操作数

赋值表达式、初始化式左操作数，为**位置上下文**，只有位置表达式能处于该上下文中，用于获取所有权，值表达式不能位于这些上下文中；

```rust
let a; //声明位置表达式a
a = Box::new(7); //a为位置表达式，在赋值表达式左侧，接收所有权
Box::new(5) = Box::new(7); //err，值表达式不能位于赋值表达式左侧
```

右操作数为**值上下文**，位置表达式和值表达式处于该上下文中，将移动所有权；

```rust
let a = Box::new(7); //值表达式Box::new(7)处于值上下文中，`Move`或`Copy`
let b = a; //位置表达式a处于值上下文中，`Move`或`Copy`
```

### 函数调用传入的参数

根据函数签名，参数为非引用类型则`Move`或`Copy`传入参数所有权；

#### 方法调用

函数调用的一种形式，方法调用的**接受者**会自动引用或解引用，使传入接受者的类型符合函数签名要求；

```rust
struct A {
    a: i32,
}
impl A {
    fn add(&self) {     //传入&self的参数借用所有权
        self.a += 1;
    }
}
fn main() {
    let mut m = A{ a: 3 };
    m.add(); // 这里m被自动引用，等同于(&m).add();
    println!("{:?}", m);
}
```

#### 方法调用的过程

Rust创建一个列表，将不断解引用接受者表达式获取的类型存入列表中，最后强制转化为无尺寸类型（如，切片）并将每一个类型`T`、`&T`和`&mut T`按顺序添加到列表中；

接着遍历列表中的每一个类型，类型的顺序是`self`、`&self`、`&mut self`、`*self`、`&(*self)`、`&mut (*self)`、强制转换的类型`T`、`&T`、`&mut T`。

在下述两个地方查找类型的方法：

- `T`本身直接实现的方法
- `T`实现的`trait`中的方法。如果`T`是类型参数，类型绑定的`trait`将优先查找，接着查找作用域中实现的`trait`方法

```rust
let n: Box<i32> = Box::new(-7); // n为Box<i32>类型
println!("{}", n.abs()); //7
// 上述过程为：
// 查找Box<i32>、&Box<i32>、&mut Box<i32>、i32、&i32、&mut i32的方法abs()，
// 在i32类型中查找到abs方法，接着调用<i32>.abs()
```

发生查找冲突时，须将接收器类型转化为适当类型避免编译错误。

### 借用操作数

位置表达式处于借用上下文中，借用所有权；

```rust
let a = Box::new(7);
&a; //值上下文中a的值不会转移
```

值表达式处于借用上下文中，表达式的值隐式赋值给临时变量，同时借用临时变量所有权，临时变量离开作用域，值将`drop`；

```rust
let a = &Box::new(7); // 获取临时变量所有权
// let tmp = Box::new(7);
// let a = &tmp;
```

### 解引用操作数

应用于指针，表明指针所指向的位置。如果表达式是`&mut T`和`*mut T`类型，且非临时变量，那么`*(&mut T)`及`*(*mut T)`可被赋值；

应用于非指针，该类型必须实现`std::ops::Deref`，那么`*x`等同于`*std::ops::Deref::deref(&x)`；

**解引用表达式不会获取所有权，也无法通过解引用表达式`Move`所有权**。

```rust
let a = &Box::new(7);
let b = *a; //err，*a处于值上下文，但无法`Move`
```

### 字段表达式操作数

字段表达式的左操作数是结构实例，若左操作数是指针，则rust**自动将指针解引用**；

注意：对结构字段的引用与对结构引用的字段是不同的类型，前者是引用类型，后者是字段的类型；

```rust
struct A {
    a: i32,
}
let m = A{ a: 3 };
let b: &i32 = &m.a //类型为&i32
let c: i32 = (&m).a //类型为i32,&m自动解引用
```

### 检查表达式

`if let`、`while let`和`match`表达式中的检查表达式是**位置上下文**，传入值表达式，`match`表达式创建临时变量储存值；传入位置表达式，不创建临时变量；

分支与检查表达式进行匹配，若有变量绑定，则`Move`或`Copy`所有权。

```rust
struct A {
    a: i32,
}
fn main() {
    let a = A { a: 3 };
    match a {
        A { a: 3 } => (), // 匹配成功，但不Move a 的所有权
        A { a: _ } => (),
    }
    match a {
        b => (), // 匹配成功，b绑定a的值，Move或Copy所有权
    }
}
```

### 闭包



## 生命周期

为避免悬垂指针，引用的生命周期应该包含在被引用对象的生命周期内，在引用的生命周期内，不允许转移对象所有权。

### 块作用域

声明在块作用域中的变量，离开作用域后被销毁，即本地变量的生命周期与块作用域的范围一致。编译器能够检查出块作用域中的引用生命周期。

```rust
let a;
{
    let b = String::new("hello");
    a = &b // err, 引用a的生命周期范围比b的生命周期长
}
let c = String::new("world");
c; // c 被move
a = &c; //err, 无法引用被move的对象
```

### 生命周期注解

对于一些**项**，如果与引用类型相关，编译器无法判断项与引用、项内引用与引用之间生命周期的关系，此时需要为项添加范型参数并为引用类型添加生命周期注解，约定项与各个引用之间生命周期的相对关系。

生命周期注解用于引用类型，`&'a T`表示`T`的引用类型，其生命周期为`a`。

在函数中运用生命周期注解

```rust
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str { //表示，函数的输入生命周期和输出生命周期至少等于a
    //
}
```

# 类型

## 一些类型

### 结构体

结构体使用`struct`关键字声明，结构体内包含不同类型的字段，结构体包含三种形式：

- 单元结构体，不包含任何字段

  ```rust
  struct A; //类单元结构体
  let a = A; //类单元结构体实例，值为()
  let b = A{}; //等同于a
  ```

- 无字段名的结构体，元组结构体

  ```rust
  struct A(i32, String, i32);
  struct A{
      0: i32,
      1: String,
      2: i32,
  } //A的另一种形式
  let a = A(3, String::from("hello world"), 4);
  let c = a.0 //使用索引，获取字段值
  ```

- 带字段名的结构体

  ```rust
  struct A {
      a: i32,
      s: String,
  } //声明结构体
  let a = 3;
  let b = A{
      a, // 传入结构值的外部变量名与字段名相同时的简写形式，等同于`a: a,`
      s: String::from("hello world"),
  };
  let c = b.a; //获取字段值
  let d = A {
      ..b,
  }; //使用b结构初始化d结构，注意`b.s`被Move
  ```

### 枚举

使用`enum`关键字声明枚举类型，枚举体包含数种可能的成员，成员类型相同，成员声明形式与结构体类似。

```rust
enum A {
    Unit, //声明一个类似单元结构体形式成员
    Copy(i32, i32, i32), //声明一个类似单元结构体成员
    Move {
        s: String,
        i: i32,
    }, //声明一个常规结构体形式成员
}
// 枚举实例化
let a = A::Unit;
let b = A::Copy(2, 3, 4);
let c = A::Move{
    s: String::from("hello"),
    i: 2,
};
// a, b, c的类型都为A
fn s(a: A) {} //可以向s中传入a, b, c
```

枚举类型底层涵盖一个*discriminant*和占最大空间的成员所需要的空间，*discriminant*对应枚举成员的生产器。所以一个无字段的成员实例，值不是`()`，这与结构体不一样。

```rust
enum A {
    Unit,
}
let a = A::Unit; //非`()`
```

### 闭包

### 函数指针

## 范型

能被参数化的项称为**泛型项**，包括：

- 函数
- 类型别名
- 结构体
- 枚举
- `trait`
- `impl`

这些项能被**类型**和**生命周期**参数化，参数被列在项名后面的`<>`中，而`impl`无项名所以`<>`直接位于`impl`关键字之后，声明周期参数必须列在类型参数之前。

泛型项的泛型参数运用于项内部的部件：

```rust
fn f<'a, T>(num: i32, plus: &'a T) -> i32 { // 声明了一个泛型函数，声明中的参数和返回值必要时应注明对应的类型和声明周期参数
    //
}
struct A<'a, T, Y> { // 声明了一个结构体，类型和声明周期参数，运用于结构体字段
    a: &'a T,
    b: Y,
}
trait A<T> {
    fn new() -> Self::T;
}
impl<T> 
```



### 生命周期

## Trait

`trait`定义一组能被实现的抽象接口，接口由三种关联的项组成：

- 函数
- 类型别名
- 常量

`trait`定义一个隐含的类型参数`Self`,表示所有实现了该接口的**类型**。`trait`也可以包含额外的类型参数，这些类型参数（包括`Self`）可以被其他的`trait`约束。

`trait`被特定的类型通过`impl`项实现，`trait`定义时，关联项可以被定义也可以不定义：如果定义了，所有的`impl`项若未**重载**关联项，则这些定义作为`impl`项的默认实现；否则，`impl`必须实现这些关联项。

泛型使用`trait`作为类型参数的绑定。`trait`可以指定类型参数，作为一个泛型`trait`。

### 关联项

**关联项**是声明在`trait`中或定义在`impl`中的项。关联项有两种：`impl`中的**定义**，`trait`项签名的**声明**。

#### 关联函数和方法

**关联函数声明**语句为关联函数定义声明一个签名，**关联函数定义**必须与声明的签名保持一致；当关联函数在`trait`中声明后，函数可以通过附加在`trait`名上的路径进行调用，一旦调用路径被替换为`<_ as Trait>::function_name`。

```rust
trait Num {
    fn from_i32(n: i32) -> Self; // 声明关联函数
}

impl Num for f64 {
    fn from_i32(n: i32) -> f64 { n as f64 } // 定义关联函数
}

// 调用关联函数
let _: f64 = Num::from_i32(42); // 通过trait名路径调用关联函数
let _: f64 = <_ as Num>::from_i32(42); // 调用关联函数
let _: f64 = <f64 as Num>::from_i32(42); // 通过转化类型为trait调用关联函数
let _: f64 = f64::from_i32(42); // 通过类型调用关联函数
```

第一参数命名为`self`的关联函数被称为**方法**,可通过方法调用操作符`.`调用。`self`参数限定为下述类型：

- `Self`
- `&Self`
- `&mut Self`
- `Box<Self>`
- `Rc<Self>`
- `Arc<Self>`
- `Pin<P>` P为上述几种类型，除了`Self`类型

可以使用简写语法来不指定类型：

```rust
struct Example;
impl Exmple {
    fn by_value_1(self: Self) {}
    fn by_value_2(self) {} // 等同于上式
    
    fn by_ref_1(self: &Self) {}
    fn by_ref_2(&self) {} // 等同于上式
    
    fn by_ref_mut_1(self: &mut Self) {}
    fn by_ref_mut_2(&mut self) {} // 等同于上式
    
    fn by_box(self: Box<Self>) {}
    fn by_rc(self: Rc<Self>) {}
    fn by_arc(self: Arc<Self>) {}
    fn by_pin(self: Pin<&Self>) {}
    fn explicit_type(self: Arc<Example>) {} // 可以使用正在实现的类型替代Self
    fn with_lifetime<'a>(self: &'a Self) {}
}
```

可以使用`mut`修饰`self`，与`mut`修饰表示符类似。

#### 关联类型

**关联类型**是与其它类型相关联的类型别名。关联类型不能在类型**固有impl**中定义，也不能在`trait`中有默认实现，即关联类型只能在`trait`中声明，在特定类型对`trait`的实现中定义。**关联类型声明**为**关联类型定义**声明一个签名。

```rust
trait AssociatedType {
    type Assoc; // 声明关联类型
}
struct Struct;
struct OtherStruct;
impl AssociatedType for Struct {
    type Assoc = OtherStruct; // 定义关联类型
}
impl OtherStruct {
    fn new() -> OtherStruct {
        OtherStruct
    }
}

fn main() {
    let _other_struct：OtherStruct = <Struct as AssociatedType>::Assoc::new(); //AssociatedType::Assoc 在 Struct的实现中为 OtherStruct 的别名
}
```

关联类型的意义，在`trait`中若需要实现`trait`的类型对_另外的类型_进行操作，而该类型需要在具体的实现中确定，那么在`trait`声明中可通过`type`声明为该类型设置占位符。

```rust
pub trait Iterator {
    type Item;
    fn next(&mut self) -> Option<Self::Item>;
}
impl Iterator for Counter { // 实现trait时，需要将Item一同实现
    type Item = u32;
    fn next(&mut self) -> Option<u32> { // 关联类型被实现中指定的类型替代
    //
    }
}
```

那么为什么不直接实现一个泛型`trait`?

#### 关联常量

```rust
trait ConstantId {
    const ID: i32; // 声明关联常量
}

struct Struct;

impl ConstantId for Struct {
    const ID: i32 = 1; // 定义关联常量
}
trait ConstantIdDefault {
    const ID: i32 = 1; // 声明一个默认关联常量
}

struct Struct;
struct OtherStruct;

impl ConstantIdDefault for Struct {} // 使用默认关联常量

impl ConstantIdDefault for OtherStruct { // 重载默认常量
    const ID: i32 = 5;
}

fn main() {
    assert_eq!(1, Struct::ID);
    assert_eq!(5, OtherStruct::ID);
}
```



### `Supertraits`

`supertraits`是某个类型在实现特定`trait`时，必须实现的`trait`。此外，如果任一泛型或`trait`对象被`trait`绑定，则该泛型或`trait`对象可以访问`supertraits`的关联项。

`supertraits`通过对特定`trait Self`的`trait`绑定进行定义，当`trait`定义在绑定中，传递`supertrait`。`trait`称为`supertrait`的`subtrait`。

```rust
trait Shape { fn area(&self) -> f64; }
trait Circle : Shape { fn radius(&self) -> f64; } //将Shape声明为Circle的supertrait

trait Circle where Self: Shape { //另一种声明supertrait的方式
    fn radius(&self) -> f64 {
        // A = pi * r^2
        // so algebraically,
        // r = sqrt(A / pi)
        (self.area() /std::f64::consts::PI).sqrt() //可直接访问supertrait的关联项area
    }
}

fn print_area_and_radius<C: Circle>(c: C) {
    // Here we call the area method from the supertrait `Shape` of `Circle`.
    println!("Area: {}", c.area()); //supertrait传递给绑定trait的泛型
    println!("Radius: {}", c.radius());
}
```

### 参数模式

函数或方法未定义体，其参数模式只能是**标识符**或**—**模式（通配符），只限于：

- _identifier_
- —（通配符）
- `&` _identifier_
- `&&` _identifier_

# 模式

# 迭代器

# 并发与异步

