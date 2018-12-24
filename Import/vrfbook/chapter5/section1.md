#put vrf into the jbok

1. 测试的时候编写测试用例，有两种：一种是直接嵌入在代码中，另一个是外部的测试用例。外部测试用例有两种互补的风格，一种是bdd（行为规格驱动）：主体-》should must can -> 描述-〉具体代码。 另一种是基于性质的的测试，forall等等whenever.这类测试写法有个好处就是方便理解。

2. package object 其实跟package 一样，可以用com.xxx.xx引用，但是他有一个好处，就是包兼容，意思就是说2.77 scala升级到 2.8时 List本来是在scala包下的，但是要放到scala.collections.immutable中去，所以用了package object scala.来实现目的 

3. 
