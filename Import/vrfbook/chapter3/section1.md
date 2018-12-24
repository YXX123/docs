# scala

纯函数：没有副作用,就是说不会有以下几种行为
* i/o
* console
* 修改对象的成员数据
* 重新赋值一个变量
* exception

> 举个例子：
> val x=new StringBuilder("hello")  x.append(" world"),这个例子中x就改变了他自己。

RT：相关透明性（一个纯函数，必须满足RT，其实没有副作用的真正含义就是RT）
是说如果一个表达式可以被他的结果完全替代，而不会有任何的不同,这个很容易验证，当你将一个表达式移动以下的时候，你发现结果完全变了，那么就是非RT的。很显然上面列举的5点都是这种情况。
```scala
var x=2+3
var m=x+y
//此处完全可以用5替代x+y中的x,而不会对结果有任何影响
var x=print("xxx is a dog")
print("xxx is a cat")
var m=x
//注意此处你将x替代掉和不替代，结果完全不同，后者先打印dog 后打印cat ,前者完全相反
```
  
为什么要用这种纯函数呢？
`模块化和重用，将计算和结果分开来`
> 举个例子：

```scala
    case class Player(name:String,score:Int)
    def printWinner(p:Player){
        println(p.name+"is the winner")
    }
    def declareWinner(p1:Player,p2:Player){
        if(p1.score>p2.score) printWinner(p1)
        else printWinner(p2)
    }
    //这种方法中declareWinner中混合了比较和展示结果这两个动作
    def printWinner(p:Player){
        println(p.name+"is the winner")
    }
    def winner(p1: Player, p2: Player): Player =
      if (p1.score > p2.score) p1 else p2
    def declareWinner(p1: Player, p2: Player): Unit =
      printWinner(winner(p1, p2)) //若产生了副作用，winner就没法用在这儿  
     //而这种方法中将declareWinner的比较逻辑和打印代码分开
    //此时这个winner的函数就可以复用了，否则没办法复用的
    val players = List(Player("Sue", 7),
                   Player("Bob", 8),
                   Player("Joe", 4))
    val p = players.reduceLeft(winner)
    printWinner(p)
```
任何函数都可以拆分成纯函数和副作用函数的组成
 