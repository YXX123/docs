# scala

>1. 什么是尾递归？：就是返回的值是一个函数本身
> 尾递归的好处：尾递归最后可以由编译器优化成循环函数，这样子就不会出现递归会出现的栈溢出问题
`示例：斐波那契函数如下，最后返回的go(n-1,acc1,acc+acc1)，就是一个尾递归`
```scala
    def fib(n:Int):Int={
    def go(n:Int,acc:Int,acc1:Int): Int ={
      n match {
        case 0=>acc
        case 1=> acc1
        case _=>go(n-1,acc1,acc+acc1)
      }
    }
    go(n,0,1)
  }
  需要注意的是：函数式编程中，经常将返回结果放到参数中，可以这么理解，斐波那契函数其实接受三个参数，n，0(第一个返回值)，1(第二个返回值)
  再看一个示例:阶乘函数,他接受一个n,1(第一返回值)，而返回值经过函数真正作用之后就能够返回。
  def factorial(n: Int): Int = {
  def go(n: Int, acc: Int): Int =
    if (n <= 0) acc
    else go(n-1, n*acc)
    go(n, 1) 
    }
```

> 2. 高阶函数：将一个函数作为参数传递给另一个函数，那么这个函数就是高阶函数
> 示例：
```scala
  def A(a:Int,f:Int=>Int):Int={
    f(a)
  }
  注意的是，其实只要根据函数的签名就能够实现这个函数的结构体了，在这个A函数中，如果函数体部分不填f(a)，那么该填什么呢？读者可以自己好好体会一下，会发现无法填入其他的函数体，这也是函数式编程保证程序错误概率下降的一个重要原因     
```

> 3. 匿名函数：函数不用写函数名
> 示例如下：
```scala
  在高阶函数定义的A 函数中，这么去调用：  A(5,x=>x+5) ,这其中的x=>x+5就是匿名函数，
  匿名函数的写法有很多种，_=>_+5,连名子都省了,x:Int=>x+5,标注x的类型，但是有时候就不用标注，因为scala的类型推导非常强
  匿名函数，其实都是
```

> 4. 函数与方法:函数是对象，而方法不是对象,在面向对象的编程中，我们很容易知道函数调用，但是如果我希望获得这个函数本身，这样子就可以将它传给高阶函数了。
举例如下：
```scala
object A{
  def a(c:Int):Int={
    "xxxx"
  }
  val func(Int=>Int)=c:Int=>{
      "xxxx"
  }
}
注意这个a就是一个方法，所以我们在使用时 A.a(5),这表示调用，但是如果就是想拿到a这个方法，怎么办？
可以 `val b=A.a _ `，通过这种方式拿到这个a本身，'A.a _'是将这个方法转为了函数。 
或者可以通过func函数，A.func拿到这个函数本身 ,函数的调用是 A.func(5)
```
> 5. 多态函数：其实就是Java中多态，C++中的模版，也就是说有些时候算法是一样的，但是类型不同，这个时候我们肯定希望一个算法就能搞定，不希望针对不同的数据类型，还要写不同的函数。
> `比如：二分查找的算法`
```scala
  def binarySearch[A](as: Array[A], key: A, gt: (A,A) => Boolean): Int = {
  @annotation.tailrec
  def go(low: Int, mid: Int, high: Int): Int = {
    if (low > high) -mid - 1
    else {
      val mid2 = (low + high) / 2
      val a = as(mid2)
      val greater = gt(a, key)
      if (!greater && !gt(key,a)) mid2
      else if (greater) go(low, mid2, mid2-1)
      else go(mid2 + 1, mid2, high)
  } }
  go(0, 0, as.length - 1)
}
注意在这个算法中，若针对Int,Double，String分别写不同的函数，那么就太麻烦了
```
`多态的一大优势是一旦函数的签名确定了，那么函数的实现也会随之确定，举例如下：`
```scala
   def partial1[A,B,C](a: A, f: (A,B) => C): B => C={
     //函数体的实现必为：
     b=>f(a,b)
     //原因是类型必须匹配，f(a,b)的返回类型是C ，b的类型是B（这是默认类型推导的结果）。
     //再详细解释一下，调用时如 partial1(5,(x:Int,y:Int)=>x+y)，可以继续分解为：y=>5+y，可以好好体会一下
```

> 6. 柯里化：比如如果一个函数有两个参数，但是我现在只拿到了一个参数，那么我可以用这个参数带入后得到了另一个只有一个参数的函数
```scala
   def curry[A,B,C](f: (A, B) => C): A => (B => C)={
     a=>b=>f(a,b)
   }
   def func(a:Int,b:Int):Int={
     a+b
   }
   //具体调用如下：
   curry(func)(5)，也就是说，curry(func)返回一个A=>(B=>C)类型的函数，然后我们给这个函数传入5这个参数，于是curry(func)(5)返回了B=>C类型的函数，这个时候我们再可以curry(func)(5)(6)，这个表达式返回了C类型的数，即 11.
```
反柯里化也一样：注意=>具有右结合性
```scala
 def uncurry[A,B,C](f: A => B => C): (A, B) => C={
   (a,b)=>f(a)(b)
 }
```
