---
title: 逆变与协变
date: 2021-05-01T00:50:44+08:00
slug: covariance_and_contravariance
categories:
tags:
    - Concept
    - Note
---

> 逆变和协变看了忘忘了看, 这次学C#顺带查阅各处总结了下...

**协变**


协变是指能够使用比原始指定的类型派生程度更大的类型.

例如Dog与Cat继承自Animal, 那么从Dog->Animal就称为协变, 那么协变有什么作用呢?

```c#
// Animal.cs
public abstract class Animal
{
}
public class Dog : Animal
{
}
```

```c#
// Main.cs
public static void Main()
{
    Dog dog = new Dog();
    Animal animal = dog;
    List<Dog> dogs = new List<Dog>();
    List<Animal> animals = dogs; // error
}
```

Dog继承自Animal, 所以Dog可以隐式的转化为Animal, 但是List<Dog>与List<Animal>之间没有继承关系, 所以无法隐式转换, 如果想要隐式转换需要如下代码:

```c#
List<Animal> animals = dogs.Select(d => (Animal)d).ToList();
```

将小狗列表中的小狗挨个显式转换为Animal.

所以C#提供了协变的语法糖, 也就是out, 意思是指该泛型可以作为输出返回.
也就是说可以用如下代码:

```c#
// Main.cs 
public static void Main() 
{ 
    Dog dog = new Dog(); 
    Animal animal = dog; 
    IEnumerable<Dog> dogs = new List<Dog>();
    IEnumerable<Animal> animals = dogs;
}

// IEnumerable
public interface IEnumerable<out T>: IEnumerable
```

因为T只能作为结果返回, 所以T不会被修改, 编译器可以帮我们进行强制转换.
 但实际上out只是上面强制转换的一个语法糖而已, 实际上反编译的代码依然进行的是强制类型转换.

```c#
IEnumerable<Animal> animals = (IEnumerable<Animal>) dogs;
```

至于为什么只能作为结果返回而不能作为输入呢?
我们假设:
- A ≼ B 意味着 A 是 B 的子类型.
- A → B 指的是以 A 为参数类型, 以 B 为返回值类型的函数类型.
- x : A 意味着 x 的类型为 A.
假设我有如下三种类型:

```
Greyhound ≼ Dog ≼ Animal
```

问题: 以下哪种类型是 Dog → Dog 的子类型呢？

1. Greyhound → Greyhound
2. Greyhound → Animal
3. Animal → Animal
4. Animal → Greyhound

我们假设 f 是一个以 Dog → Dog 为参数的函数, 那么可以这样假设: f : (Dog → Dog) → String

那么容易回答上述问题:
1.  不是, 因为传入的类型可能是狗的其它自类型但不是灰狗, 这样也是符合函数签名的, 但是答案1就不符合了.
2.  同上
3.  不是, 因为有可能 f 在调用完参数之后让它的返回值(Animal)狗叫, 但是并非所有的动物都会狗叫, 所以同样存在符合函数签名但不是答案4的结果.
4.  是的, 所有的Dog都是Animal, 所以传进去的任何Dog都是符合的, 而所有的Greyhoud都是狗, 也都可以狗叫.

这样就可以得到我们最初的答案了---为什么只能作为结果返回而不能作为参数输入, 因为所有的Dog都是Animal, 但是如果作为参数的话, 那么不是所有的Animal都是Dog, 就有可能往IEnumerable<Dog>里传一个Cat, 但这又是符合IEnumerable<Animal>签名的, 所以是不行的.

而由上面的例子也可以得到, 与协变相反的就是逆变, 也就是超类可以作为泛型参数而不能作为输出结果, 简单的公式为:
(Animal → Greyhound) ≼ (Dog → Dog)

解释为: 我们允许一个函数类型中, 返回值类型是协变的, 而参数类型是逆变的. 返回值类型是协变的, 意思是 A ≼ B 就意味着 (T → A) ≼ (T → B) . 参数类型是逆变的, 意思是 A ≼ B 就意味着 (B → T) ≼ (A → T)  A 和 B 的位置颠倒过来了, 通俗点协变就是返回值可以是超类, 而逆变则是参数可以是子类.