#scala 数据结构
假设有如下类型
```scala
sealed trait List[+A]
case object Nil extends List[Nothing]
case class Cons[+A](head: A, tail: List[A]) extends List[A]
object List {
  def sum(ints: List[Int]): Int = ints match {
    case Nil => 0
    case Cons(x,xs) => x + sum(xs)
}
  def product(ds: List[Double]): Double = ds match {
    case Nil => 1.0
    case Cons(0.0, _) => 0.0
    case Cons(x,xs) => x * product(xs)
}
  def apply[A](as: A*): List[A] =
    if (as.isEmpty) Nil
    else Cons(as.head, apply(as.tail: _*))
  val example = Cons(1, Cons(2, Cons(3, Nil)))
  val example2 = List(1,2,3)
  val total = sum(example)
}
```
> 解释如下：
1. trait 是抽象的interface,sealed 表示关于List的实现都在当前文件中，
case 修饰object和class有以下特性：
```scala
1. 修饰class时，默认也给了class的伴生对象，即相当于同时声明了class和object
2. 接受模式匹配，也就是match 
3. 构造参数都是public的
4. 默认实现了hashcode,equals,serializable
```
> 2. 协变类型,可以看到List[+A]，这个+号的意思是,若B是A的子类型，那么List[B]也是List[A]的子类型

> 3. 模式匹配,其实就是if else语句，嫌弃这种分叉的写法太浪费代码，所以有了这么一种简洁的语法，
```scala
def sum(ints: List[Int]): Int = ints match {
    case Nil => 0
    case Cons(x,xs) => x + sum(xs)
}  
这是在方法中使用，更常规的使用方法是：
l match {
  case xx=> xx
  case xxx=> xxx
}
其中 l是前面声明的一个变量
```

> 4. 可变参数函数, 下面的A\* 就是可变参数，as的数组大小可以是0或者更多，这个跟Java 中String ...a,这种声明是一样的，A\*的实际类型是Seq[A]
```
def apply[A](as: A*): List[A] =
```
> 数据分享,我们说Scala中我们希望是纯函数，纯函数返回的数据是不可变的，immutable这种，那么当我们
```scala
  Cons(1,xs)//看看代码中example对象的，Cons是一个类，照理说是要new Cons(1,xs)的，但是由于有object中的apply函数存在，所以前面不用加new
  //这个xs是一个List对象，此时Cons对象并没有将xs复制一份，而是直接将1链接到了xs这个对象，这就是数据分享
```
> 5. 一般化，如果发现某些函数的算法是一致的，那么可以将算法提取出来：
> 举例：
```scala
  def sum(ints: List[Int]): Int = ints match {
    case Nil => 0
    case Cons(x,xs) => x + sum(xs)
  }
  def product(ds: List[Double]): Double = ds match {
    case Nil => 1.0
    case Cons(0.0, _) => 0.0
    case Cons(x,xs) => x * product(xs)
  }
  //这个加法和乘法，找出共性，分辨差异可以看出，Nil时的返回值不同，+ 和 \* 不同,其他都一样,如下，z是初始值，f是函数
  def foldRight[A,B](l: List[A], z: B)(f: (A, B) => B): B =
    l match {
    case Nil => z
    case Cons(x, xs) => f(x, foldRight(xs, z)(f))
  }
  def sum2(l: List[Int]) =
    foldRight(l, 0.0)(_ + _)
  def product2(l: List[Double]) =
    foldRight(l, 1.0)(_ * _)
  //f(x, foldRight(xs, z)(f))可以看出这个函数一直对l的tail进行计算，计算的结果再拿来与head x进行f运算
```
> 6. foldRight：其实可以这么去理解他，用z去取代Nil,用f去取代Cons，表示z是最初始化的结果，而f的第二个元素就是最终的结果，例如：下面求list长度的函数,acc就是最终的结果，这个结果的初始值是0（z），因为f是Cons的替代，(\_, acc)，第一个\_就是每个具体元素，表示每次当遇到case Cons时，初始值为0的acc都会acc+1.
```
 def length[A](l: List[A]): Int = {
    foldRight(l, 0)((_, acc) => acc + 1)
  }
```
> 有些人可能会把foldRight写成这样子
```
  def foldRight[A,B](l: List[A], z: B)(f: (A, B) => B): B =
    l match {
    case Nil => z
    case Cons(x, xs) => foldRight(xs,f(x, z))(f)
    //虽然这个变成了尾递归，但是其实他是先算的f(x,z)，也就是先从头开始对每个元素进行计算，这样子意思就变掉了，我们看一个append函数
    //def append[A](l1:List[A],l2:List[A]):List[A]={
    //  foldRight(l1,l2)((x,xs)=>Cons(x,xs))
    //}
    这时候可以看到[1,2,3] append[4,5,6] 会变成；[3,2,1,4,5,6]
}
``` 

7.foldLeft: foldRight有一个问题，就是他不是尾递归的，所以无法被编译器变为循环结构，所以可能产生栈溢出。但foldLeft不一样：
```
 def foldLeft[A, B](l: List[A], z: B)(f: (B, A) => B): B = l
  match       
   {
    case Nil => z
    case Cons(h, t) => foldLeft(t, f(z, h))(f)
    // 此处展示了一个错误的用法：case Cons(h,t) => f(foldLeft(t,z)(f),h)
  }
  //此处的f变成了B在左侧，A在右侧，其实这个顺序没有关系，也可以是：
  def foldLeft[A, B](l: List[A], z: B)(f: (A,B) => B): B = l
  match       
   {
    case Nil => z
    case Cons(h, t) => foldLeft(t, f(h, z))(f)
  }
  //length函数变成了
  def length[A](l: List[A]): Int = {
    foldLeft(l, 0)((acc, _) => acc + 1)
  }
```

 8. 为什么用foldRight或者foldLeft结合其他的一下简单函数，就可以实现基本上所有的功能？：原因在于fold类的是对List的两种基本类型的一个分别计算，而List只有这两个基本类型，所以对这两个基本类型的操作都可以抽象出去成f.
 
 9. 难题：用foldLeft实现foldRight?暂时掠过！
 
 10. 下面将通过flapMap说明一个难以理解的疑惑点：
 ```
   def flatMap[A, B](l: List[A])(f: A => List[B]): List[B] = {
    foldRight(l, Nil: List[B])((h, t) => append(f(h), t))
  }
  //flatMap会将l展开之后再拼接在一起，此时看到其实append是将两个List连接在一起的函数，但是foldRight(l, Nil: List[B])((h, t) => append(f(h), t)
中的(h,t)其实代表的是一个元素h和一个最终的结果t,记住t并不是tail。理解了这一点，我们就很容易判断了。
 ```
 
adt：代数数据类型，其实就是首先有一些基本元素，然后基本元素组成的集合就是adt

二叉树类型的操作：
























