# 纯函数式状态
1. 含义就是说本来会产生副作用的函数，现在通过返回新的对象的方式来来达到RT
：比如
```scala
    var rng=new scala.util.Random
    rng.nextInt 
    rng.nextInt //这两个nextInt 返回的是不同的值
    但是我希望是RT的
    // 这个nextInt并不修改原先的RNG
     trait RNG{
      def nextInt:(Int,RNG)
    }
```
2. 如何写出可测试性好的代码？事实上，一般程序应当分离为依赖外部的调用，和不依赖外部的调用，第一种不用单测，因为没有任何逻辑，第二种要单侧，因为有函数逻辑。
```java
    (1)代码语义化，比如：
    if(state.equal(5)){
        //code
    }这句话没有含义，而且万一state为null就会产npe
    应该改为： //这种就有寓意了，这种称为业务点
    public class OrderStateUtil {
    public static isOrderPaid() {
        return Integer.valueOf(State.ISPAID).equals(state);
    }
    }
    (2)独立逻辑分离出来：独立逻辑就是说纯函数，有相同的输入就有相同的输出
    这样才更容易编写单元测试
    (3)分离实例状态：pickURl依赖一个fileServer,所以编写单元测试的时候就会发现得启动整个FileService，这是有问题的，可以将rand抽象在一个randutil中，
    public class FileService {
    @Value("${file.server}")
    private String fileServer;
    /**
     * 随机选取上传服务器
     * @return 上传服务器URL
     */
    private String pickUrl(){
        String urlStr = fileServer;
        String[] urlArr = urlStr.split(",");
        int idx = rand.nextInt(2);
        return urlArr[idx].trim();
    }
    }
    (4)分离外部服务调用,
    public Map<String, Object> searchForSelect(@RequestParam(value = "k", required = false) String title,
                                               @RequestParam(value = "page", defaultValue = "1") Integer page,
                                               @RequestParam(value = "rows", defaultValue = "10") Integer pageSize) {
        CreativeQuery query = buildCreativeQuery(title, page, pageSize);
        return searchForSelect2(query,
                               (q) -> creativeService.search(q),
                               (q) -> creativeService.count(q));
    }

    public Map<String, Object> searchForSelect2(CreativeQuery query,
                                               Function<CreativeQuery, List<CreativeDO>> getListFunc,
                                               Function<CreativeQuery, Integer> getTotalFunc) {
        List<CreativeDO> creativeDTOs = getListFunc.apply(query);
        Integer total = getTotalFunc.apply(query);
        Map<String, Object> map = new HashMap<String, Object>();
        map.put("rows", (null == creativeDTOs) ? new ArrayList<CreativeDO>() : creativeDTOs);
        map.put("total", (null == total) ? 0 : total);
        return map;
    }
    /*
     * NOTE: can be placed in class QueryBuilder
     */
    public CreativeQuery buildCreativeQuery(String title, Integer page, Integer pageSize) {
        CreativeQuery query = new CreativeQuery();
        query.setTitle(title);
        query.setPageNum(page);
        query.setPageSize(pageSize);
        return query;
    }
    public class CreativeControllerTest {
  CreativeController controller = new CreativeController();
  @Test
  public void testSearchForSelect2() {
    CreativeQuery creativeQuery = controller.buildCreativeQuery("haha", 1, 20);
    Map<String, Object> result = controller.searchForSelect2(creativeQuery,
                                                            (q) -> null , (q)-> 0);
    Assert.assertEquals(0, ((List)result.get("rows")).size());
    Assert.assertEquals(0, ((Integer)result.get("total")).intValue());

  }
}
//注意代码的测试testSearchForSelect2中的，测试lamda就非常的方便，注意searchForSelect2(CreativeQuery query,                Function<CreativeQuery, List<CreativeDO>> getListFunc,Function<CreativeQuery, Integer> getTotalFunc)
//中的function中result，我们可以在测试用例中构建很多List<CreativeDo>类型的参数。
   (5)表达与执行分离：注意if中选择发货子是逻辑，而getBean其实是执行，前者需要单元测试，而后者无需单测。
   public Deliverer getDeliverInstance(DeliveryContext deliveryContext, ExpressParam params) {
 
    if (periodDeliverCondtion1) {
      LogUtils.info(log, "periodDeliverer for {}", params);
      return (Deliverer) applicationContext.getBean("periodDeliverer");
    }
 
    if(periodDeliverCondtion2){
      LogUtils.info(log, "periodDeliverer for {}", params);
      return (Deliverer) applicationContext.getBean("periodDeliverer");
    }
 
    if (fenxiaoDelivererCondition) {
      LogUtils.info(log, "fenxiaoDeliverer for {}", params);
      return (Deliverer) applicationContext.getBean("fenxiaoDeliverer");
    }
    if (giftDelivererCondition) {
      LogUtils.info(log, "giftDeliverer for {}", params);
      return (Deliverer) applicationContext.getBean("giftDeliverer");
    }
    if (localDelivererCondition) {
      LogUtils.info(log, "localDeliverr for {}", JsonUtils.toJson(order));
      return (Deliverer) applicationContext.getBean("localDeliverer");
    }
    LogUtils.info(log, "normalDeliverer for {}", params);
    return (Deliverer) applicationContext.getBean("normalDeliverer");
  }
```
本质上函数原子由这些功能构成，前三者是逻辑u部分，想要单侧的。
构建参数
判断条件是否满足
组装数据
调用服务查询数据
调用服务执行操作
3. 写函数其实很简单，要不你就用组合子，要不你就直接分类讨论，分类讨论分为两种，一种是recursive 另一种是tail recursive的。组合子可以根据自己的需要进行构造。
```scala
  def ints(count:Int,rng:RNG):(List[Int],RNG)={
    if(count==0) (List(),rng)
    else{
      val (i,r1)=rng.nextInt
      val (i2,r2)=ints(count-1,r1)
      (i::i2,r2)
    }
  }
```
4. 状态转换是一种专门的类型，有特殊的用途：S=>(A,S)，一个状态转化为另一个新的数据和状态的组合。而这个东西直接写的话太麻烦了
```scala
比如有一个map:这种写法太麻烦了
  def map[A,B](s:RNG=>(A,RNG))(f:A=>B):(RNG=>(B,RNG))={
    rng=>{
      val (i1,r1)=s(rng)
      (f(i1),r1)
    }
  }
  //一种新的写法是将RNG=>(A,RNG)写个别名
  type Rand[A]= RNG=>(A,RNG)，于是变成了
   def map[A,B](s:Rand[A])(f:A=>B):(Rand[B])={
    rng=>{
      val (i1,r1)=s(rng)
      (f(i1),r1)
    }
  } 
```
可以看到 RNG 这中可以抽象成人意类型的S，甚至可以用一个class来表示

5.  
 
