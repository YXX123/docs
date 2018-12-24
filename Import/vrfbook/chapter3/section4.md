#scala 异常处理

1. 假如异常如下：
```scala
def failingFn(i: Int): Int = {
  val x: Int = throw new Exception("fail!")
  try {
val y = 42 + 5
x+ y }
  catch { case e: Exception => 43 }
}
//可以看到RT问题，就是说x在调用的地方换成x的真实值，程序的结果是不一样的，一个会报异常，一个会返回43.
def mean(xs: Seq[Double]): Double =
  if (xs.isEmpty)
    throw new ArithmeticException("mean of empty list!")
  else xs.sum / xs.length
```
这个函数如果传递给高阶函数，会有问题是高级函数必须去处理这个异常，而从函数的签名可以看出，这个函数压根儿就没希望抛出异常，此时会有一个巨大问题：当我们调用任何函数的时候需要明确知道这个函数的具体的返回值中会不会抛出异常，这个太费事儿了。

2. 有没有解决办法呢？针对mean求平均值这个函数又如下变种
```
(1) 返回一个特殊的值
  def mean(xs: Seq[Double]): Double =
  if (xs.isEmpty)
    Double.NaN
  else xs.sum / xs.length
(2) 调用者手动传入一个值
def mean(xs: Seq[Double],onEmpty:Double): Double =
if (xs.isEmpty)
onEmpty
else xs.sum / xs.length
```
>可以看出上面两个方法，有问题：
(1)就是对于多态函数来说，这个特殊值该怎么设置呢？null 或者0 ,这没法提前知道，(2)二是调用者可能不太想知道被调用函数的具体错误值，因为他希望只管理他自己的东西就可以了，(3)三是：假使被调用函数出现可错误，调用者可能希望灵活的去处置他，而不是任凭这个错误消失了，或者希望abort

其实解决这些问题有一个好方法，问题的本质就是说调用者希望可以有处理或者不处理的权力，还有调用者如果处理那么它希望可以简单的处理。
如果可以设定一个简单的抽象类型，去统一管理，会非常有优势
3. 综上所述：现在定义Option如下
```scala
//用option去做包装，然后异常时返回None这个错误值，这样子调用者就可以自由选择如何处理
sealed trait Option[+A]
case class Some[+A](get: A) extends Option[A]
case object None extends Option[Nothing]
def mean(xs: Seq[Double]): Option[Double] =
if (xs.isEmpty)
  None
else Some(xs.sum / xs.length)
//mean可能定义在Option的伴生对象中，读者可以去试试看这个mean定义是否解决了1,2中提到的各种弊端
```
4. 部分函数和全函数，部分函数：就是一个函数的返回值可能不是函数签名中定义的类型，比如 :他就返回了一个异常，而非全部的Double，全函数则相反
```
def mean(xs: Seq[Double]): Double =
  if (xs.isEmpty)
    throw new ArithmeticException("mean of empty list!")
  else xs.sum / xs.length
```
5. 在class中定义函数和在object中定义的区别？
在List中，我们定义了很多函数，但是大多数想调用时必须这样子
```
  List.foldRight(l:List，f)
  //可以看出object中定义的必须外部传入一个list，而不能直接l.foldRight,但是class中定义的就可以，因为class中定义的函数本身就有class的this的引用,如下：
  sealed trait Option[+A] {
  def map[B](f: A => B): Option[B]
  def flatMap[B](f: A => Option[B]): Option[B]
  def getOrElse[B >: A](default: => B): B
  def orElse[B >: A](ob: => Option[B]): Option[B]
  def filter(f: A => Boolean): Option[A]
}
//调用时,Some(1).map(_+1)，直接是这个obj. 点出来调用
变种： obj.fn(arg1) 等同于 obj fn arg1 ,而在object中定义的函数会变为：fn(obj,arg1)
```
6. 按名调用和按值调用的区别：
我们知道scala中函数与方法的区别
```scala
  def max_1(x:Int,x:Int):Int={
    if (x>y) x
    else y
  }//方法
  def max_2=(x:Int,x:Int)=> {
    if (x>y) x
    else y
  }//函数
  如果我们在高阶函数中传入max_2,那么max_2只有在调用时才会被求值，而不像max_1那样被求值后才被传给高级函数,原因在于max_2其实是一个值,是一个new Function2,max_2只拿到了这个函数对象，并没有被调用。
  而按名调用的原理跟这个差不多，如下：
  def and(x: Boolean, y: Boolean) = x && y
  and(false, s.contains("horance"))
  传入函数时变成了：
  def and(x: () => Boolean, y: () => Boolean) = x() && y()
  and(() => false, () => s.contains("horance"))
  //发现一堆括号，麻烦了许多，所以scala将这些括号去掉之后，就变成了按名调用
```


7. map与flatMap的区别，map是A->B 的转换，flatMap是A->Option[B]的转换，List中也是这样，List.flatMap是A->List[B]的转化
我们会很纳闷这个flatMap的作用：
```
  def variance(xs:Seq[Double]):Option[Double]={
    mean(xs) flatMap (m=>mean(xs.map(x=>Math.pow(x-m,2))))
  }
```

8. for yield:比如你有一个高阶函数，你可以先从for中得到几个参数，然后一次性使用他们
```scala
def pattern(s: String): Option[Pattern] =
    try {
      Some(Pattern.compile(s))
    } catch {
      case e: PatternSyntaxException => None
    }
def mkMatcher(pat: String): Option[String => Boolean] =
      pattern(pat) map (p => (s: String) => p.matcher(s).matches)
def bothMatch(pat: String, pat2: String, s: String):Option[Boolean] =
      for {  
        f <- mkMatcher(pat)
        g <- mkMatcher(pat2)
    } yield f(s) && g(s)
    //可以看到对多 个pattern求值时，可以先在for中得到多个matcher，然后在yield中统一调用，<-表示先取出里面的值，最后f(s)&&g(s)又会讲结果进行包装成Option类型的东西
```
9. 函数到底是什么：其实本质上依然没变，就是完成特定功能的，你可以想想要完成Java函数第一步先做什么，第二步再做什么，注意的地方有如下：
```
  *. Scala 中函数有两种基本思路：使用map等的基本函数可以完成一些映射，或者使用case进行分门别类的运算
  *. 一个函数的功能是有1种，那么高阶函数可以看成1*N种功能，所以可以实现的功能要复杂的多，比如我现在有一个map函数，那么我给map函数传递的函数，其实才是真正起做用的。
  *. 函数要完成的功能不会变，基于含义进行的功能，mkName(name).map2(mkAge(age))(Person(_,_))
  *. 可以先拿到一个数据，然后再拿另一个数据，依次拿到这些值，然后去操作,
  mkName(name).map2(mkAge(age))(Person(_,_)),先拿到name,再去拿到age,然后再去构造成一个完整的Person,这其中map2是取得基本的讲两个值进行映射的功能。
```

10. 为什么一个函数中需要B:>A这种使用方法？
以Option的getOrElse为例,这里的A与Option中的A是属于同一类，但是并不是同一个，这里的A相当于Option中的A的拷贝，但是不是同一个对象，
```scala
  def getOrElse[B:>A](default:=>B):B={
    this match{
    case None=>default
    case Some(a)=>a
    }
  }
```


