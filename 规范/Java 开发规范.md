# 一、编程规约
## (一) 命名风格

1. 【强制】代码中的命名不能使用下划线或美元符号开始或者结束。 
   * 反例：`_name / __name / $name / name_ / name$ / name__`



2. 【强制】命名严禁使用拼音与英文混合的方式，更不允许直接使用中文的方式。注意，纯拼音命名方式更要避免采用。

   * 正例：`ali / alibaba / taobao / hangzhou` 等国际通用的名称，可视同英文。
   * 反例：`DaZhePromotion [打折] / String fw[福娃] / int 某变量 = 3`

   

4. 【强制】类名使用 UpperCamelCase 风格，但以下情形例外：`DO / BO / DTO / VO / AO / PO / UID` 等。 

   * 正例：`ForceCode / UserDO / HtmlDTO`
   * 反例：`forcecode / UserDo / HTMLDto`

   

5. 【强制】方法名、参数名、成员变量、局部变量都统一使用 lowerCamelCase 风格。

   * 正例： `localValue / getHttpMessage() / inputUserId`

   

6. 【强制】常量命名全部大写，单词间用下划线隔开，力求语义表达完整清楚，不要嫌名字长。

   * 正例：`MAX_STOCK_COUNT / CACHE_EXPIRED_TIME`
   * 反例：`MAX_COUNT / EXPIRED_TIME`

   

7. 【强制】抽象类命名使用Abstract或Base开头；异常类命名使用Exception结尾；测试类命名以它要测试的类的名称开始，以Test结尾。



8. 【强制】POJO类中的任何布尔类型的变量，都不要加is前缀，否则部分框架解析会引起序列化错误。

   * 反例：布尔类型变量XXX的判断方法一般是isXXX()，因此一些框架在反向解析该变量时，会自动找到isXXX() 方法从而解析出变量名XXX。如果变量名定义为 isXXX，则会导致部分框架反解析变量名错误。
   * 说明：mysql 对于是与否的字段应采用 is_xxx 的命名方式，因此需要在 `<resultMap>` 设置从is_xxx到xxx的映射关系。

   

9. 【强制】包名统一使用小写，点分隔符之间有且仅有一个自然语义的英语单词。包名统一使用单数形式，但是类名如果有复数含义，类名可以使用复数形式。 

   * 正例：应用工具类包名为 `com.alibaba.ei.kunlun.aap.util`，类名为 MessageUtils。

   

10. 【强制】避免在子父类的成员变量之间、不同代码块的局部变量之间采用完全相同的命名，使可理解性降低。

    * 反例：

      ```java
      public class ConfusingName {
          public int stock;
          // 非setter/getter的参数名称，不允许与本类成员变量同名
          public void get(String alibaba) {
              if (condition) {
                  final int money = 666;
                  // ...
              }
              for (int i = 0; i < 10; i++) {
                  // 在同一方法体中，不允许与其它代码块中的money命名相同
                  final int money = 15978;
                  // ...
              }
          }
      }
      class Son extends ConfusingName {
          // 不允许与父类的成员变量名称相同
          public int stock;
      }
      ```

11. 【强制】杜绝完全不规范的缩写，导致可读性降低。 
    * 反例：AbstractClass 缩写成 AbsClass。



12. 【推荐】如果模块、接口、类、方法使用了设计模式，在命名时需体现出具体模式。

    * 正例： `OrderFactory / LoginProxy / ResourceObserver`

    

13. 【推荐】接口类中的方法和属性不要加任何修饰符号（public 也不要加），保持代码的简洁性，并加上有效的Javadoc 注释。尽量不要在接口里定义变量，如果一定要定义变量，确定与接口方法相关，并且是整个应用的基础常量。 
    * 正例：接口方法签名 void commit(); 接口基础常量 String COMPANY = "alibaba"; 
    * 反例：接口方法定义 public abstract void f(); 
    * 说明：JDK8 中接口允许有默认实现，那么这个 default 方法，是对所有实现类都有价值的默认实现。



14. 【强制】对于 Service 和 DAO 类，基于 SOA 的理念，暴露出来的服务一定是接口，内部的实现类用Impl的后缀与接口区别。

    * 正例：CacheServiceImpl 实现 CacheService 接口。

    

15. 【推荐】如果是形容能力的接口名称，取对应的形容词为接口名（通常是–able的形容词）。
    * 正例：AbstractTranslator 实现 Translatable接口。



16. 【参考】枚举类名带上 Enum 后缀，枚举成员名称需要全大写，单词间用下划线隔开。

* 说明：枚举其实就是特殊的常量类，且构造方法被默认强制是私有。
* 正例：枚举名字为 ProcessStatusEnum 的成员名称：SUCCESS / UNKNOWN_REASON。



17. 【参考】各层命名规约： 
    * Service/DAO层方法命名规约：
      * 获取单个对象的方法用get做前缀。
      * 获取多个对象的方法用list做前缀，复数结尾，如：listObjects。
      * 获取统计值的方法用count做前缀。
      * 插入的方法用save/insert做前缀。
      * 删除的方法用remove/delete做前缀。
      * 修改的方法用update做前缀。
    * 领域模型命名规约：
      * 数据对象：xxxDO，xxx即为数据表名。
      * 数据传输对象：xxxDTO，xxx为业务领域相关的名称。
      * 展示对象：xxxVO，xxx一般为网页名称。
      * POJO是DO/DTO/BO/VO的统称，禁止命名成xxxPOJO。



## (二) 常量定义

1. 【强制】不允许任何魔法值（即未经预先定义的常量）直接出现在代码中。



2. 【强制】在 long 或者 Long 赋值时，数值后使用大写字母L，不能是小写字母l，小写容易跟数字混淆，造成误解。



3. 【推荐】不要使用一个常量类维护所有常量，要按常量功能进行归类，分开维护。
   * 正例：缓存相关常量放在类 CacheConsts 下；系统配置相关常量放在类 SystemConfigConsts 下。



4. 【推荐】常量的复用层次有五层：跨应用共享常量、应用内共享常量、子工程内共享常量、包内共享常量、类内共享常量。

   * 跨应用共享常量：放置在二方库中，通常是 client.jar 中的 constant 目录下。
   * 应用内共享常量：放置在一方库中，通常是子模块中的 constant 目录下。
   * 子工程内部共享常量：即在当前子工程的 constant 目录下。
   * 包内共享常量：即在当前包下单独的 constant 目录下。
   * 类内共享常量：直接在类内部 private static final 定义。

   

5. 【推荐】如果变量值仅在一个固定范围内变化用 enum 类型来定义。

  

## (三) 代码格式

1. 【强制】如果是大括号内为空，则简洁地写成{}即可，大括号中间无需换行和空格；如果是非空代码块则： 

   ```java
   // {}无内容
   public void method() {}
   // {}有内容
   // {前有空格，{后换行
   // }前换行，}后还有else等代码则不换行，表示终止的右大括号后必须换行
   public void method() {
       System.out.println("ok");
   }
   ```

2. 【强制】左小括号和右边相邻字符之间不出现空格；右小括号和左边相邻字符之间也不出现空格；而左大括号前需要加空格。

   ```java
   if (a == 1) {
   	System.out.println("ok");
   }
   ```

3. 【强制】if/for/while/switch/do等保留字与括号之间都必须加空格。



4. 【强制】任何二目、三目运算符的左右两边都需要加一个空格。 说明：包括赋值运算符=、逻辑运算符&&、加减乘除符号等。



5. 【强制】采用4个空格缩进，禁止使用Tab字符。
   * 说明：如果使用Tab缩进，必须设置1个Tab为4个空格。IDEA设置Tab为4个空格时，请勿勾选Use tab character；而在Eclipse中，必须勾选insert spaces for tabs。

* 正例：

  ```java
  public static void main(String[] args) {
      // 缩进4个空格
      String say = "hello";
      // 运算符的左右必须有一个空格
      int flag = 0;
      // 关键词if与括号之间必须有一个空格，括号内的f与左括号，0与右括号不需要空格
      if (flag == 0) {
          System.out.println(say);
      }
      // 左大括号前加空格且不换行；左大括号后换行
      if (flag == 1) {
          System.out.println("world");
          // 右大括号前换行，右大括号后有else，不用换行
      } else {
          System.out.println("ok");
          // 在右大括号后直接结束，则必须换行
      }
  }
  ```

6. 【强制】注释的双斜线与注释内容之间有且仅有一个空格。

6. 【强制】在进行类型强制转换时，右括号与强制转换值之间不需要任何空格隔开。

* 正例：

  ```java
  double first = 3.2d; int second = (int)first + 2;
  ```

8. 【强制】单行字符数限制不超过120个，超出需要换行，换行时遵循如下原则： 

  * 第二行相对第一行缩进4个空格，从第三行开始，不再继续缩进，参考示例。

  * 运算符与下文一起换行。

  * 方法调用的点符号与下文一起换行。

  * 方法调用中的多个参数需要换行时，在逗号后进行。

  * 在括号前不要换行。

  * 正例：

    ```java
    StringBuilder sb = new StringBuilder();
    // 超过120个字符的情况下，换行缩进4个空格，并且方法前的点号一起换行
    sb.append("yang").append("hao")...
        .append("chen")...
        .append("chen")...
        .append("chen");
    ```

9. 【强制】方法参数在定义和传入时，多个参数逗号后面必须加空格。

  * 正例：

    ```java
    public void method(args1, args2, args3);
    ```

10. 【强制】IDE 的 text file encoding 设置为 UTF-8，IDE 中文件的换行符使用Unix格式，不要使用Windows格式。 

11. 【推荐】单个方法的总行数不超过80行。



## (四) OOP规约

1. 【强制】静态变量或静态方法使用类名直接访问。



2. 【强制】所有的覆写方法，必须加@Override注解。



3. 【强制】相同参数类型，相同业务含义，才可以使用Java的可变参数，避免使用Object。 说明：可变参数必须放置在参数列表的最后。（建议开发者尽量不用可变参数编程） 

   * 正例：

     ```java
     public List<User> listUsers(String type, Long... ids) {...}
     ```

4. 【强制】外部正在调用或者二方库依赖的接口，不允许修改方法签名，避免对接口调用方产生影响，如要修改应新增接口，并对旧的接口加@Deprecated，标识该接口已过时。@Deprecated注解标识的接口为过时接口，需要添加说明告知调用者应采用的新接口或者新服务是什么。



5. 【强制】不能使用过时的类或方法。 



6. 【强制】Object的equals方法容易抛空指针异常，应使用常量或确定有值的对象来调用 equals。
   * 正例："test".equals(object); 
   * 反例：object.equals("test"); 
   * 说明：推荐使用JDK7引入的工具类java.util.Objects#equals(Object a, Object b)



7. 【强制】所有整型包装类对象之间值的比较，全部使用equals方法比较。 
   * 说明：对于Integer var = ? 在-128至127之间的赋值，Integer对象是在 IntegerCache.cache 产生，会复用已有对象，这个区间内的 Integer 值可以直接使用 == 进行判断，但是这个区间之外的所有数据，都会在堆上产生，并不会复用已有对象。因此推荐使用 equals 方法进行判断。



8. 【强制】任何货币金额，均以最小货币单位且整型类型来进行存储。



9. 【强制】浮点数之间的等值判断，基本数据类型不能用 == 来比较，包装数据类型不能用 equals。

   * 说明：浮点数比较存在小数的精确度偏差，导致等值判断看似一样，实际因为该偏差导致判断为不相等。

   * 正例：

     ```java
     // 指定一个误差范围，两个浮点数的差值在此范围之内，则认为是相等的。 float a = 1.0F - 0.9F;
     float b = 0.9F - 0.8F;
     float diff = 1e-6F;
     if (Math.abs(a - b) < diff) {
         System.out.println("true");
     } 
     
     // 使用BigDecimal来定义值，再进行浮点数的运算操作。
     BigDecimal a = new BigDecimal("1.0");
     BigDecimal b = new BigDecimal("0.9");
     BigDecimal c = new BigDecimal("0.8");
     BigDecimal x = a.subtract(b);
     BigDecimal y = b.subtract(c);
     if (x.compareTo(y) == 0) {
         System.out.println("true");
     }
     ```

10. 【强制】如上所示 BigDecimal 的等值比较应使用 compareTo() 方法，而不是equals()方法。 

   * 说明：equals() 方法会比较值和精度（1.0与1.00返回结果为false），而 compareTo() 则会忽略精度。

11. 【强制】定义数据对象DO类时，属性类型要与数据库字段类型相匹配。 

    * 正例：数据库字段的bigint必须与类属性的Long类型相对应。 

12. 【强制】禁止使用构造方法 BigDecimal(double) 的方式把 double 值转化为BigDecimal对象。

    * 说明：BigDecimal(double) 存在精度损失风险，在精确计算或值比较的场景中可能会导致业务逻辑异常。如：BigDecimal g = new BigDecimal(0.1F); 实际的存储值为：0.10000000149 
    * 正例：优先推荐入参为String的构造方法，或使用 BigDecimal 的 valueOf方法，此方法内部其实执行了Double 的 toString，而 Double 的 toString 按 double 的实际能表达的精度对尾数进行了截断。

13. 关于基本数据类型与包装数据类型的使用标准如下：

    * 【强制】所有的POJO类属性必须使用包装数据类型。POJO类属性没有初值是提醒使用者在需要使用时，必须自己显式地进行赋值，任何NPE问题，或者入库检查，都由使用者来保证。
    * 【强制】RPC方法的返回值和参数必须使用包装数据类型。
    * 【推荐】所有的局部变量使用基本数据类型。
    * 正例：数据库的查询结果可能是 null，因为自动拆箱，用基本数据类型接收有NPE风险。
    * 反例：某业务的交易报表上显示成交总额涨跌情况，即正负x%，x为基本数据类型，调用的RPC服务，调用不成功时，返回的是默认值，页面显示为0%，这是不合理的，应该显示成中划线-。所以包装数据类型的null值，能够表示额外的信息，如：远程调用失败，异常退出。

14. 【强制】定义DO/DTO/VO等POJO类时，不要设定任何属性默认值。 

    * 反例：POJO类的 createTime 默认值为new Date()，但是这个属性在数据提取时并没有置入具体值，在更新其它字段时又附带更新了此字段，导致创建时间被修改成当前时间。

15. 【强制】序列化类新增属性时，请不要修改 serialVersionUID 字段，避免反序列失败；如果完全不兼容升级，避免反序列化混乱，那么请修改 serialVersionUID 值。

    * 说明：注意serialVersionUID不一致会抛出序列化运行时异常。

16. 【强制】构造方法里面禁止加入任何业务逻辑，如果有初始化逻辑，请放在init方法中。

17. 【强制】POJO类必须写 toString 方法。

    * 说明：在方法执行抛出异常时，可以直接调用POJO的toString()方法打印其属性值，便于排查问题。

18. 【强制】禁止在POJO类中，同时存在对应属性 xxx 的 isXxx() 和 getXxx() 方法。

    * 说明：框架在调用属性xxx的提取方法时，并不能确定哪个方法一定是被优先调用到的。

19. 【推荐】使用索引访问用 String 的 split 方法得到的数组时，需做最后一个分隔符后有无内容的检查，否则会有抛 IndexOutOfBoundsException 的风险。

    * 说明：如 `"a,b,c,,"` 分割后实际只有三个内容，而不是四个。

20. 【推荐】当一个类有多个构造方法，或者多个同名方法，这些方法应该按顺序放置在一起，便于阅读，此条规则优先于下一条。

21. 【推荐】 类内方法定义的顺序依次是：公有方法或保护方法 > 私有方法 > getter / setter 方法。

22. 【推荐】setter方法中，参数名称与类成员变量名称一致。在getter/setter方法中，不要增加业务逻辑，增加排查问题的难度。

    * 反例：

      ```java
      private Integer data;
      
      public void setData(Integer val) {
          this.data = val;
      }
      
      public Integer getData () {
          if (condition) {
              return this.data + 100;
          } else {
              return this.data - 100;
          }
      }
      ```

      

23. 【推荐】循环体内，字符串的连接方式，使用 StringBuilder 的 append 方法进行扩展。 

    * 说明：下例中，反编译出的字节码文件显示每次循环都会 new 出一个 StringBuilder 对象，然后进行append 操作，最后通过 toString 方法返回 String 对象，造成内存资源浪费。

    * 反例：

      ```java
      String str = "start";
      for (int i = 0; i < 100; i++) {
          str = str + "hello";
      }
      ```

      

24. 【推荐】final可以声明类、成员变量、方法、以及本地变量，下列情况使用final关键字： 

    * 不允许被继承的类，如：String类。
    * 不允许修改引用的域对象，如：POJO类的域变量。
    * 不允许被覆写的方法，如：POJO类的setter方法。
    * 不允许运行过程中重新赋值的局部变量。
    * 避免上下文重复使用一个变量，使用final关键字可以强制重新定义一个变量，方便更好地进行重构。

25. 【推荐】慎用Object的clone方法来拷贝对象。

    * 说明：对象clone方法默认是浅拷贝，若想实现深拷贝，需覆写clone方法实现域对象的深度遍历式拷贝。

26. 【推荐】类成员与方法访问控制从严： 

    * 如果不允许外部直接通过new来创建对象，那么构造方法必须是private。
    * 工具类不允许有public或default构造方法。
    * 类非static成员变量并且与子类共享，必须是protected。
    * 类非static成员变量并且仅在本类使用，必须是private。
    * 类static成员变量如果仅在本类使用，必须是private。
    * 若是static成员变量，考虑是否为final。
    * 类成员方法只供类内部调用，必须是private。
    * 类成员方法只对继承类公开，那么限制为protected。



## (五) 日期时间

1. 【强制】日期格式化时，传入pattern中表示年份统一使用小写的y。

   * 说明：日期格式化时，yyyy表示当天所在的年，而大写的YYYY代表是week in which year（JDK7之后引入的概念），意思是当天所在的周属于的年份，一周从周日开始，周六结束，只要本周跨年，返回的YYYY就是下一年。
   * 正例：表示日期和时间的格式如下所示： new SimpleDateFormat("yyyy-MM-dd HH:mm:ss") 

   

2. 【强制】在日期格式中分清楚大写的M和小写的m，大写的H和小写的h分别指代的意义。 

   * 说明：
     * 表示月份是大写的M；
     * 表示分钟则是小写的m；
     * 24小时制的是大写的H；
     * 12小时制的则是小写的h。



3. 【强制】获取当前毫秒数：System.currentTimeMillis();  而不是new Date().getTime()。
   * 说明：如果想获取更加精确的纳秒级时间值，使用 System.nanoTime 的方式。在JDK8中，针对统计时间等场景，推荐使用 Instant 类。



4. 【强制】不允许在程序任何地方中使用：1）java.sql.Date。 2）java.sql.Time。 3）java.sql.Timestamp。 说明：第1个不记录时间，getHours()抛出异常；第2个不记录日期，getYear()抛出异常；第3个在构造方法super((time/1000)*1000)，在Timestamp 属性fastTime和nanos分别存储秒和纳秒信息。
   反例： java.util.Date.after(Date)进行时间比较时，当入参是java.sql.Timestamp时，会触发JDK BUG(JDK9已修复)，可能导致比较时的意外结果。



5. 【强制】不要在程序中写死一年为365天，避免在公历闰年时出现日期转换错误或程序逻辑错误。

   * 正例：

     ```java
     // 获取今年的天数 
     int daysOfThisYear = LocalDate.now().lengthOfYear(); 
     // 获取指定某年的天数 
     LocalDate.of(2011, 1, 1).lengthOfYear();
     ```

   * 反例：

     ```java
     // 第一种情况：在闰年366天时，出现数组越界异常 
     int[] dayArray = new int[365]; 
     // 第二种情况：一年有效期的会员制，今年1月26日注册，硬编码365返回的却是1月25日 
     Calendar calendar = Calendar.getInstance(); 
     calendar.set(2020, 1, 26); 
     calendar.add(Calendar.DATE, 365);
     ```

6. 【推荐】避免公历闰年2月问题。闰年的2月份有29天，一年后的那一天不可能是2月29日。

7. 【推荐】使用枚举值来指代月份。如果使用数字，注意Date，Calendar等日期相关类的月份month取值在0-11之间。 

   * 正例： Calendar.JANUARY，Calendar.FEBRUARY，Calendar.MARCH 等来指代相应月份来进行传参或比较。



## (六) 集合处理

1. 【强制】关于hashCode和equals的处理，遵循如下规则：

   * 只要覆写equals，就必须覆写hashCode。
   * 因为Set存储的是不重复的对象，依据 hashCode 和 equals 进行判断，所以Set存储的对象必须覆写这两种方法。
   * 如果自定义对象作为Map的键，那么必须覆写hashCode和equals。
   * 说明：String 因为覆写了 hashCode 和 equals 方法，所以可以愉快地将 String 对象作为 key 来使用。

   

2. 【强制】判断所有集合内部的元素是否为空，使用 isEmpty() 方法，而不是size()==0的方式。 说明：在某些集合中，前者的时间复杂度为O(1)，而且可读性更好。

3. 【强制】在使用 java.util.stream.Collectors 类的 toMap() 方法转为Map集合时，一定要使用含有参数类型为BinaryOperator，参数名为 mergeFunction 的方法，否则当出现相同 key 值时会抛出 IllegalStateException异常。 

  * 说明：参数mergeFunction的作用是当出现key重复时，自定义对value的处理策略。 

  * 正例：

    ```java
    List<Pair<String, Double>> pairArrayList = new ArrayList<>(3);
    pairArrayList.add(new Pair<>("version", 12.10));
    pairArrayList.add(new Pair<>("version", 12.19));
    pairArrayList.add(new Pair<>("version", 6.28));
    Map<String, Double> map = pairArrayList.stream().collect(
        // 生成的map集合中只有一个键值对：{version=6.28}
        Collectors.toMap(Pair::getKey, Pair::getValue, (v1, v2) -> v2)); 
    ```

  * 反例：

    ```java
    String[] departments = new String[] {"iERP", "iERP", "EIBU"};
    // 抛出IllegalStateException异常
    Map<Integer, String> map = Arrays.stream(departments)
    .collect(Collectors.toMap(String::hashCode, str -> str)); 
    ```



4. 【强制】在使用java.util.stream.Collectors 类的 toMap() 方法转为Map集合时，一定要注意当 value 为 null时会抛NPE异常。

   * 说明：在 java.util.HashMap 的merge方法里会进行如下的判断：

     ```java
     if (value == null || remappingFunction == null)
         throw new NullPointerException(); 
     ```



5. 【强制】ArrayList 的 subList 结果不可强转成 ArrayList，否则会抛出 ClassCastException 异常：java.util.RandomAccessSubList cannot be cast to java.util.ArrayList。

   * 说明：subList() 返回的是 ArrayList 的内部类 SubList，并不是 ArrayList 本身，而是 ArrayList 的一个视图，因此 SubList 的所有操作最终会反映到原列表上，原列表的操作也可能会影响 SubList。

   

6. 【强制】使用Map的方法 keySet() / values() / entrySet() 返回集合对象时，不可以对其进行添加元素操作，否则会抛出 UnsupportedOperationException 异常。



7. 【强制】Collections类返回的对象，如：emptyList() / singletonList() 等都是 immutable list，不可对其进行添加或者删除元素的操作。 调用方一旦进行了添加或删除元素的操作，就会触发UnsupportedOperationException 异常。



8. 【强制】在 subList 场景中，高度注意对父集合元素的增加或删除，均会导致子列表的遍历、增加、删除产生ConcurrentModificationException 异常。 



9. 【强制】使用集合转数组的方法，必须使用集合的 toArray (T[] array)，传入的是类型完全一致、长度为0的空数组。 直接使用 toArray 无参方法存在问题，此方法返回值只能是Object[]类，若强转其它类型数组将出现ClassCastException错误。 

   * 说明：使用toArray带参方法，数组空间大小的length：
     * 等于0，动态创建与size相同的数组，性能最好。
     * 大于0但小于size，重新创建大小等于size的数组，增加GC负担。
     *  等于size，在高并发情况下，数组创建完成之后，size正在变大的情况下，负面影响与2相同。
     * 大于size，空间浪费，且在size处插入null值，存在NPE隐患。

   

10. 【强制】在使用 Collection 接口任何实现类的 addAll() 方法时，都要对输入的集合参数进行NPE判断。

    * 说明：在ArrayList#addAll方法的第一行代码即 Object[] a = c.toArray(); 其中c为输入集合参数，如果为null，则直接抛出异常。

    

11. 【强制】使用工具类 Arrays.asList() 把数组转换成集合时，不能使用其修改集合相关的方法，它的add/remove/clear方法会抛出UnsupportedOperationException异常。 

    * 说明：asList 的返回对象是一个Arrays内部类，并没有实现集合的修改方法。Arrays.asList 体现的是适配器模式，只是转换接口，后台的数据仍是数组。 

    

12. 【强制】泛型通配符<? extends T>来接收返回的数据，此写法的泛型集合不能使用add方法，而<? super T>不能使用get方法，两者在接口调用赋值的场景中容易出错。



13. 【强制】在无泛型限制定义的集合赋值给泛型限制的集合时，在使用集合元素时，需要进行 instanceof 判断，避免抛出 ClassCastException 异常。 

    * 反例：

      ```java
      List<String> generics = null; 
      List notGenerics = new ArrayList(10); 
      notGenerics.add(new Object()); 
      notGenerics.add(new Integer(1)); 
      generics = notGenerics; 
      // 此处抛出ClassCastException异常 
      String string = generics.get(0);
      ```

      

14. 【强制】不要在foreach循环里进行元素的remove/add操作。remove元素请使用Iterator方式，如果并发操作，需要对Iterator对象加锁。
    
15. 【强制】在JDK7版本及以上，Comparator 实现类要满足如下三个条件，不然 Arrays.sort，Collections.sort会抛 IllegalArgumentException 异常。

    * 说明：三个条件如下：
      * x，y的比较结果和y，x的比较结果相反。
      * x>y，y>z，则x>z。
      * x=y，则x，z比较结果和y，z比较结果相同。 

    

16. 【推荐】集合泛型定义时，在JDK7及以上，使用diamond语法或全省略。

    * 正例：

      ```java
      // diamond方式，即<>
      HashMap<String, String> userCache = new HashMap<>(16);
      // 全省略方式
      ArrayList<User> users = new ArrayList(10);
      ```



17. 【推荐】集合初始化时，指定集合初始值大小，从而避免多次调用 resize() 方法，影响性能。 



18. 【推荐】使用 entrySet 遍历 Map 类集合 KV，而不是 keySet 方式进行遍历。



19. 【推荐】高度注意Map类集合K/V能不能存储null值的情况，如下表格：

    | 集合类            | Key          | Value        | Super       | 说明                   |
    | ----------------- | ------------ | ------------ | ----------- | ---------------------- |
    | Hashtable         | 不允许为null | 不允许为null | Dictionary  | 线程安全               |
    | ConcurrentHashMap | 不允许为null | 不允许为null | AbstractMap | 锁分段技术（JDK8:CAS） |
    | TreeMap           | 不允许为null | 允许为null   | AbstractMap | 线程不安全             |
    | HashMap           | 允许为null   | 允许为null   | AbstractMap | 线程不安全             |

    

20. 【参考】合理利用好集合的有序性(sort)和稳定性(order)，避免集合的无序性(unsort)和不稳定性(unorder)带来的负面影响。 
    * 说明：有序性是指遍历的结果是按某种比较规则依次排列的。稳定性指集合每次遍历的元素次序是一定的。如：ArrayList是order/unsort；HashMap是unorder/unsort；TreeSet是order/sort。



21. 【参考】利用Set元素唯一的特性，可以快速对一个集合进行去重操作，避免使用 List 的 contains() 进行遍历去重或者判断包含操作。



## (七) 并发处理

1. 【强制】获取单例对象需要保证线程安全，其中的方法也要保证线程安全。

   * 说明：资源驱动类、工具类、单例工厂类都需要注意。

   

2. 【强制】创建线程或线程池时请指定有意义的线程名称，方便出错时回溯。



3. 【强制】线程资源必须通过线程池提供，不允许在应用中自行显式创建线程。

   * 说明：线程池的好处是减少在创建和销毁线程上所消耗的时间以及系统资源的开销，解决资源不足的问题。如果不使用线程池，有可能造成系统创建大量同类线程而导致消耗完内存或者“过度切换”的问题。

   

4. 【强制】线程池不允许使用 Executors 去创建，而是通过 ThreadPoolExecutor 的方式，这样的处理方式让写的同学更加明确线程池的运行规则，规避资源耗尽的风险。
   * 说明：Executors返回的线程池对象的弊端如下：
     * FixedThreadPool 和 SingleThreadPool： 允许的请求队列长度为 Integer.MAX_VALUE，可能会堆积大量的请求，从而导致OOM。
     * CachedThreadPool： 允许的创建线程数量为 Integer.MAX_VALUE，可能会创建大量的线程，从而导致OOM。

5. 【强制】SimpleDateFormat 是线程不安全的类，一般不要定义为 static 变量，如果定义为 static，必须加锁，或者使用 DateUtils 工具类。 正例：注意线程安全，使用 DateUtils。亦推荐如下处理：

   ```java
   private static final ThreadLocal<DateFormat> df = new ThreadLocal<DateFormat>() {
       @Override
       protected DateFormat initialValue() {
           return new SimpleDateFormat("yyyy-MM-dd");
       }
   };
   ```

   说明：如果是 JDK8 的应用，可以使用 Instant 代替 Date，LocalDateTime 代替 Calendar，DateTimeFormatter 代替 SimpleDateFormat。

   

6. 【强制】必须回收自定义的 ThreadLocal 变量，尤其在线程池场景下，线程经常会被复用，如果不清理自定义的 ThreadLocal 变量，可能会影响后续业务逻辑和造成内存泄露等问题。尽量在代理中使用 try-finally 块进行回收。 

  * 正例：

    ```java
    objectThreadLocal.set(userInfo);
    try {
        // ...
    } finally {
        objectThreadLocal.remove();
    }
    ```

    

7. 【强制】高并发时，同步调用应该去考量锁的性能损耗。能用无锁数据结构，就不要用锁；能锁区块，就不要锁整个方法体；能用对象锁，就不要用类锁。 

   * 说明：尽可能使加锁的代码块工作量尽可能的小，避免在锁代码块中调用RPC方法。



8. 【强制】对多个资源、数据库表、对象同时加锁时，需要保持一致的加锁顺序，否则可能会造成死锁。 
   * 说明：线程一需要对表A、B、C依次全部加锁后才可以进行更新操作，那么线程二的加锁顺序也必须是A、B、C，否则可能出现死锁。



9. 【强制】在使用阻塞等待获取锁的方式中，必须在try代码块之外，并且在加锁方法与try代码块之间没有任何可能抛出异常的方法调用，避免加锁成功后，在 finally 中无法解锁。

  * 说明一：如果在 lock 方法与try代码块之间的方法调用抛出异常，那么无法解锁，造成其它线程无法成功获取锁。
  * 说明二：如果 lock 方法在try代码块之内，可能由于其它方法抛出异常，导致在finally代码块中，unlock对未加锁的对象解锁，它会调用 AQS 的 tryRelease 方法（取决于具体实现类），抛出IllegalMonitorStateException 异常。
  * 说明三：在 Lock 对象的 lock 方法实现中可能抛出 unchecked 异常，产生的后果与说明二相同。 

  * 正例：

  ```java
  Lock lock = new XxxLock();
  // ...
  lock.lock();
  try {
      doSomething();
      doOthers();
  } finally {
      lock.unlock();
  } 
  ```

  * 反例：

  ```java
  Lock lock = new XxxLock();
  // ...
  try {
      // 如果此处抛出异常，则直接执行finally代码块
      doSomething();
      // 无论加锁是否成功，finally代码块都会执行
      lock.lock();
      doOthers();
  } finally {
      lock.unlock();
  } 
  ```

  

  

10. 【强制】在使用尝试机制来获取锁的方式中，进入业务代码块之前，必须先判断当前线程是否持有锁。锁的释放规则与锁的阻塞等待方式相同。

    * 说明：Lock 对象的 unlock 方法在执行时，它会调用 AQS 的 tryRelease 方法（取决于具体实现类），如果当前线程不持有锁，则抛出 IllegalMonitorStateException异常。 

    * 正例：

      ```java
      Lock lock = new XxxLock();
      // ...
      boolean isLocked = lock.tryLock();
      if (isLocked) {
          try {
              doSomething();
              doOthers();
          } finally {
              lock.unlock();
          }
      }
      ```

      

11. 【强制】并发修改同一记录时，避免更新丢失，需要加锁。要么在应用层加锁，要么在缓存加锁，要么在数据库层使用乐观锁，使用version作为更新依据。 说明：如果每次访问冲突概率小于20%，推荐使用乐观锁，否则使用悲观锁。乐观锁的重试次数不得小于3次。

    

12. 【强制】多线程并行处理定时任务时，Timer 运行多个 TimeTask 时，只要其中之一没有捕获抛出的异常，其它任务便会自动终止运行，使用 ScheduledExecutorService 则没有这个问题。 

    

13. 【推荐】资金相关的金融敏感信息，使用悲观锁策略。

    * 说明：乐观锁在获得锁的同时已经完成了更新操作，校验逻辑容易出现漏洞，另外，乐观锁对冲突的解决策略有较复杂的要求，处理不当容易造成系统压力或数据异常，所以资金相关的金融敏感信息不建议使用乐观锁更新。
    * 正例：悲观锁遵循一锁、二判、三更新、四释放的原则。

    

14. 【推荐】使用 CountDownLatch 进行异步转同步操作，每个线程退出前必须调用 countDown 方法，线程执行代码注意catch异常，确保 countDown 方法被执行到，避免主线程无法执行至 await 方法，直到超时才返回结果。 

    * 说明：注意，子线程抛出异常堆栈，不能在主线程 try-catch 到。

    

15. 【推荐】避免Random实例被多线程使用，虽然共享该实例是线程安全的，但会因竞争同一 seed 导致的性能下降。

    * 说明：Random 实例包括 java.util.Random 的实例或者 Math.random() 的方式。 

    * 正例：在 JDK7 之后，可以直接使用 API ThreadLocalRandom，而在 JDK7 之前，需要编码保证每个线程持有一个单独的 Random 实例。

      

16. 【推荐】通过双重检查锁（double-checked locking）（在并发场景下）存在延迟初始化的优化问题隐患，推荐解决方案中较为简单一种（适用于JDK5及以上版本），将目标属性声明为 volatile 型，比如将 helper 的属性声明修改为 `private volatile Helper helper = null;`。

    * 正例：

      ```java
      public class LazyInitDemo {
          private volatile Helper helper = null;
          public Helper getHelper() {
              if (helper == null) {
                  synchronized (this) {
                      if (helper == null) { helper = new Helper(); }
                  }
              }
              return helper;
          }
          // other methods and fields...
      }
      ```

      

17. 【参考】volatile解决多线程内存不可见问题。对于一写多读，是可以解决变量同步问题，但是如果多写，同样无法解决线程安全问题。

    * 说明：如果是count++操作，使用如下类实现：AtomicInteger count = new AtomicInteger(); count.addAndGet(1);  如果是JDK8，推荐使用 LongAdder 对象，比 AtomicLong 性能更好（减少乐观锁的重试次数）。

    

18. 【参考】HashMap在容量不够进行resize时由于高并发可能出现死链，导致CPU飙升，在开发过程中注意规避此风险。



19. 【参考】ThreadLocal对象使用static修饰，ThreadLocal无法解决共享对象的更新问题。
    * 说明：这个变量是针对一个线程内所有操作共享的，所以设置为静态变量，所有此类实例共享此静态变量，也就是说在类第一次被使用时装载，只分配一块存储空间，所有此类的对象(只要是这个线程内定义的)都可以操控这个变量。



## (八) 控制语句

1. 【强制】在一个switch块内，每个case要么通过continue/break/return等来终止，要么注释说明程序将继续执行到哪一个case为止；在一个switch块内，都必须包含一个default 语句并且放在最后，即使它什么代码也没有。

   * 说明：注意break是退出switch语句块，而return是退出方法体。

   

2. 【强制】当switch括号内的变量类型为String并且此变量为外部参数时，必须先进行null判断。 

  

3. 【强制】在 if/else/for/while/do 语句中必须使用大括号。

  * 说明：即使只有一行代码，也禁止不采用大括号的编码方式。

  

4. 【强制】三目运算符condition? 表达式1 : 表达式2中，高度注意表达式1和2在类型对齐时，可能抛出因自动拆箱导致的NPE异常。 

  * 说明：以下两种场景会触发类型对齐的拆箱操作：

    * 表达式1或表达式2的值只要有一个是原始类型。
    * 表达式1或表达式2的值的类型不一致，会强制拆箱升级成表示范围更大的那个类型。

  *  反例：

    ```java
    Integer a = 1;
    Integer b = 2;
    Integer c = null;
    Boolean flag = false;
    // a*b的结果是int类型，那么c会强制拆箱成int类型，抛出NPE异常
    Integer result=(flag? a*b : c); 
    ```

    

5. 【强制】在高并发场景中，避免使用”等于”判断作为中断或退出的条件。 

   * 说明：如果并发控制没有处理好，容易产生等值判断被“击穿”的情况，使用大于或小于的区间判断条件来代替。
   * 反例：判断剩余奖品数量等于0时，终止发放奖品，但因为并发处理错误导致奖品数量瞬间变成了负数，这样的话，活动无法终止。

   

6. 【推荐】当某个方法的代码总行数超过10行时，return / throw 等中断逻辑的右大括号后均需要加一个空行。 



7. 【推荐】表达异常的分支时，少用if-else方式，这种方式可以改写成：

   ```java
   if (condition) {
   ...
   return obj;
   }
   // 接着写else的业务逻辑代码;
   ```

   * 说明：如果非使用if()...else if()...else...方式表达逻辑，避免后续代码维护困难，请勿超过3层。
   * 正例：超过3层的 if-else 的逻辑判断代码可以使用卫语句、策略模式、状态模式等来实现。

   

8. 【推荐】除常用方法（如getXxx/isXxx）等外，不要在条件判断中执行其它复杂的语句，将复杂逻辑判断的结果赋值给一个有意义的布尔变量名，以提高可读性。

   * 说明：很多 if 语句内的逻辑表达式相当复杂，与、或、取反混合运算，甚至各种方法纵深调用，理解成本非常高。如果赋值一个非常好理解的布尔变量名字，则是件令人爽心悦目的事情。

   * 正例：

     ```java
     // 伪代码如下
     final boolean existed = (file.open(fileName, "w") != null) && (...) || (...);
     if (existed) {
     ...
     }
     ```

   

9. 【推荐】不要在其它表达式（尤其是条件表达式）中，插入赋值语句。 赋值语句需要清晰地单独成为一行。

   * 反例：

     ```java
     public Lock getLock(boolean fair) {
         // 算术表达式中出现赋值操作，容易忽略count值已经被改变
         threshold = (count = Integer.MAX_VALUE) - 1;
         // 条件表达式中出现赋值操作，容易误认为是sync==fair
         return (sync = fair) ? new FairSync() : new NonfairSync();
     }
     ```

     

10. 【推荐】循环体中的语句要考量性能，以下操作尽量移至循环体外处理，如定义对象、变量、获取数据库连接，进行不必要的try-catch操作（这个try-catch是否可以移至循环体外）。 

    

11. 【推荐】避免采用取反逻辑运算符。 



12. 【推荐】公开接口需要进行入参保护，尤其是批量操作的接口。
    * 反例：某业务系统，提供一个用户批量查询的接口，API文档上有说最多查多少个，但接口实现上没做任何保护，导致调用方传了一个1000的用户id数组过来后，查询信息后，内存爆了。



13. 【参考】下列情形，需要进行参数校验：
    * 调用频次低的方法。 
    * 执行时间开销很大的方法。此情形中，参数校验时间几乎可以忽略不计，但如果因为参数错误导致 中间执行回退，或者错误，那得不偿失。
    * 需要极高稳定性和可用性的方法。
    * 对外提供的开放接口，不管是RPC/API/HTTP接口。
    * 敏感权限入口。



14. 【参考】下列情形，不需要进行参数校验：
    * 极有可能被循环调用的方法。但在方法说明里必须注明外部参数检查。
    * 底层调用频度比较高的方法。毕竟是像纯净水过滤的最后一道，参数错误不太可能到底层才会暴露问题。一般DAO层与Service层都在同一个应用中，部署在同一台服务器中，所以DAO的参数校验，可以省略。 
    * 被声明成private只会被自己代码所调用的方法，如果能够确定调用方法的代码传入参数已经做过检查或者肯定不会有问题，此时可以不校验参数。



## (九) 注释规约

1. 【强制】类、类属性、类方法的注释必须使用Javadoc规范，使用 `/**内容*/` 格式，不得使用 `// xxx` 方式。 



2. 【强制】所有的抽象方法（包括接口中的方法）必须要用 Javadoc 注释、除了返回值、参数、异常说明外，还必须指出该方法做什么事情，实现什么功能。 
   * 说明：对子类的实现要求，或者调用注意事项，请一并说明。



3. 【强制】所有的类都必须添加创建者和创建日期。
   * 说明：在设置模板时，注意IDEA的@author为`${USER}`，而eclipse的@author为`${user}`，大小写有区别，而日期的设置统一为yyyy/MM/dd的格式。 



4. 【强制】方法内部单行注释，在被注释语句上方另起一行，使用 `//` 注释。方法内部多行注释使用 `/* */` 注释，注意与代码对齐。



5. 【强制】所有的枚举类型字段必须要有注释，说明每个数据项的用途。



6. 【推荐】与其“半吊子”英文来注释，不如用中文注释把问题说清楚。专有名词与关键字保持英文原文即可。 



7. 【推荐】代码修改的同时，注释也要进行相应的修改，尤其是参数、返回值、异常、核心逻辑等的修改。

   

8. 【推荐】在类中删除未使用的任何字段、方法、内部类；在方法中删除未使用的任何参数声明与内部变量。



9. 【参考】谨慎注释掉代码。在上方详细说明，而不是简单地注释掉。如果无用，则删除。 

   * 说明：代码被注释掉有两种可能性：
     * 后续会恢复此段代码逻辑。
     * 永久不用。前者如果没有备注信息，难以知晓注释动机。建议直接删掉即可，假如需要查阅历史代码，登录代码仓库即可。

   

10. 【参考】特殊注释标记，请注明标记人与标记时间。注意及时处理这些标记，通过标记扫描，经常清理此类标记。线上故障有时候就是来源于这些标记处的代码。

    * 待办事宜（TODO）:（标记人，标记时间，[预计处理时间]） 表示需要实现，但目前还未实现的功能。
    * 错误，不能工作（FIXME）:（标记人，标记时间，[预计处理时间]） 在注释中用FIXME标记某代码是错误的，而且不能工作，需要及时纠正的情况。



## (十) 前后端规约

1. 【强制】前后端交互的API，需要明确协议、域名、路径、请求方法、请求内容、状态码、响应体。 

  * 说明：
    * 协议：生产环境必须使用 HTTPS。
    * 路径：每一个 API 需对应一个路径，表示API具体的请求地址：代表一种资源
      * 只能为名词，推荐使用复数，不能为动词，请求方法已经表达动作意义。
      * URL 路径不能使用大写，单词如果需要分隔，统一使用下划线。
      * 路径禁止携带表示请求内容类型的后缀，比如".json",".xml"，通过accept头表达即可。 
    * 请求方法：对具体操作的定义，常见的请求方法如下：
      * GET：从服务器取出资源。
      * POST：在服务器新建一个资源。
      * PUT：在服务器更新资源。
      * DELETE：从服务器删除资源。
    * 请求内容：URL带的参数必须无敏感信息或符合安全要求；body里带参数时必须设置Content-Type。
    * 响应体：响应体 body 可放置多种数据类型，由 Content-Type 头来确定。

  

2. 【强制】前后端数据列表相关的接口返回，如果为空，则返回空数组[]或空集合{}。

  * 说明：此条约定有利于数据层面上的协作更加高效，减少前端很多琐碎的null判断。

  

3. 【强制】服务端发生错误时，返回给前端的响应信息必须包含HTTP状态码，errorCode、errorMessage、用户提示信息四个部分。



4. 【强制】在前后端交互的JSON格式数据中，所有的 key 必须为小写字母开始的lowerCamelCase风格，符合英文表达习惯，且表意完整。



5. 【强制】errorMessage 是前后端错误追踪机制的体现，可以在前端输出到type="hidden"文字类控件中，或者用户端的日志中，帮助我们快速地定位出问题。



6. 【强制】对于需要使用超大整数的场景，服务端一律使用String字符串类型返回，禁止使用Long类型。 

   * 说明：Java 服务端如果直接返回 Long 整型数据给前端，JS 会自动转换为 Number 类型（注：此类型为双精度浮点数，表示原理与取值范围等同于 Java 中的Double）。Long 类型能表示的最大值是2的63次方-1，在取值范围之内，超过2的53次方 (9007199254740992) 的数值转化为 JS 的 Number时，有些数值会有精度损失。

   

7. 【强制】HTTP请求通过URL传递参数时，不能超过2048字节。
   * 说明：不同浏览器对于URL的最大长度限制略有不同，并且对超出最大长度的处理逻辑也有差异，2048字节是取所有浏览器的最小值。

8. 【强制】HTTP请求通过body传递内容时，必须控制长度，超出最大长度后，后端解析会出错。 

   * 说明：nginx 默认限制是 1 MB，tomcat默认限制为 2MB，当确实有业务需要传较大内容时，可以通过调大服务器端的限制。

   

9.  【强制】在翻页场景中，用户输入参数的小于1，则前端返回第一页参数给后端；后端发现用户输入的参数大于总页数，直接返回最后一页。 



10. 【强制】服务器内部重定向必须使用 forward；外部重定向地址必须使用URL统一代理模块生成，否则会因线上采用 HTTPS 协议而导致浏览器提示“不安全”，并且还会带来URL维护不一致的问题。



11. 【推荐】服务器返回信息必须被标记是否可以缓存，如果缓存，客户端可能会重用之前的请求结果。 说明：缓存有利于减少交互次数，减少交互的平均延迟。

    * 正例：http 1.1中，s-maxage 告诉服务器进行缓存，时间单位为秒，用法如下:

      ```java
      response.setHeader("Cache-Control", "s-maxage=" + cacheSeconds);
      ```

    

12. 【推荐】服务端返回的数据，使用 JSON 格式而非 XML。 



13. 【推荐】前后端的时间格式统一为"yyyy-MM-dd HH:mm:ss"，统一为GMT。



14. 【参考】在接口路径中不要加入版本号，版本控制在HTTP头信息中体现，有利于向前兼容。 



## (十一) 其他

1. 【强制】在使用正则表达式时，利用好其预编译功能，可以有效加快正则匹配速度。 

   * 说明：不要在方法体内定义：Pattern pattern = Pattern.compile(“规则”);

   

2. 【强制】避免用Apache Beanutils进行属性的copy。 

   * 说明：Apache BeanUtils性能较差，可以使用其他方案比如Spring BeanUtils, Cglib BeanCopier，注意均是浅拷贝。

   

3. 【强制】注意 Math.random() 这个方法返回是double类型，注意取值的范围 0≤x<1（能够取到零值，注意除零异常），如果想获取整数类型的随机数，不要将x放大10的若干倍然后取整，直接使用Random对象的nextInt 或者 nextLong方法。



4. 【推荐】任何数据结构的构造或初始化，都应指定大小，避免数据结构无限增长吃光内存。



5. 【推荐】及时清理不再使用的代码段或配置信息。

   * 说明：对于垃圾代码或过时配置，坚决清理干净，避免程序过度臃肿，代码冗余。

   * 正例：对于暂时被注释掉，后续可能恢复使用的代码片断，在注释代码上方，统一规定使用三个斜杠(///)来说明注释掉代码的理由。如：

     ```java
     public static void hello() {
         /// 业务方通知活动暂停
         // Business business = new Business();
         // business.active();
         System.out.println("it's finished");
     }
     ```

     

# 二、异常日志

## (一) 异常处理

1. 【强制】Java 类库中定义的可以通过预检查方式规避的 RuntimeException 异常不应该通过 catch 的方式来处理，比如：NullPointerException，IndexOutOfBoundsException 等等。 

   * 说明：无法通过预检查的异常除外，比如，在解析字符串形式的数字时，可能存在数字格式错误，不得不通过 catch NumberFormatException 来实现。 

   * 正例：

     ```java
     if (obj != null) {...} 
     ```

   * 反例：

     ```java
     try { obj.method(); } catch (NullPointerException e) {…} 
     ```

   

2. 【强制】catch 时请分清稳定代码和非稳定代码，稳定代码指的是无论如何不会出错的代码。对于非稳定代码的 catch 尽可能进行区分异常类型，再做对应的异常处理。

   * 说明：对大段代码进行try-catch，使程序无法根据不同的异常做出正确的应激反应，也不利于定位问题，这是一种不负责任的表现。

3. 【强制】捕获异常是为了处理它，不要捕获了却什么都不处理而抛弃之，如果不想处理它，请将该异常抛给它的调用者。最外层的业务使用者，必须处理异常，将其转化为用户可以理解的内容。



4. 【强制】事务场景中，抛出异常被catch后，如果需要回滚，一定要注意手动回滚事务。



5. 【强制】finally 块必须对资源对象、流对象进行关闭，有异常也要做try-catch。 

   * 说明：如果JDK7及以上，可以使用try-with-resources方式。 

   

6. 【强制】不要在finally块中使用return。 

   * 说明：try块中的return语句执行成功后，并不马上返回，而是继续执行finally块中的语句，如果此处存在return语句，则在此直接返回，无情丢弃掉try块中的返回点。

   * 反例：

     ```java
     private int x = 0;
     public int checkReturn() {
         try {
             // x等于1，此处不返回
             return ++x;
         } finally {
             // 返回的结果是2
             return ++x;
         }
     }
     ```

   

7. 【强制】捕获异常与抛异常，必须是完全匹配，或者捕获异常是抛异常的父类。 说明：如果预期对方抛的是绣球，实际接到的是铅球，就会产生意外情况。 



8. 【强制】在调用RPC、二方包、或动态生成类的相关方法时，捕捉异常必须使用Throwable类来进行拦截。 

   * 说明：通过反射机制来调用方法，如果找不到方法，抛出 NoSuchMethodException。什么情况会抛出NoSuchMethodError 呢？二方包在类冲突时，仲裁机制可能导致引入非预期的版本使类的方法签名不匹配，或者在字节码修改框架（比如：ASM）动态创建或修改类时，修改了相应的方法签名。这些情况，即使代码编译期是正确的，但在代码运行期时，会抛出NoSuchMethodError。

   

9. 【推荐】方法的返回值可以为 null，不强制返回空集合，或者空对象等，必须添加注释充分说明什么情况下会返回 null 值。 
   * 说明：本手册明确防止 NPE 是调用者的责任。即使被调用方法返回空集合或者空对象，对调用者来说，也并非高枕无忧，必须考虑到远程调用失败、序列化失败、运行时异常等场景返回null的情况。



10. 【推荐】防止NPE，是程序员的基本修养，注意NPE产生的场景：
    * 返回类型为基本数据类型，return包装数据类型的对象时，自动拆箱有可能产生NPE。 
    * 数据库的查询结果可能为null。
    * 集合里的元素即使isNotEmpty，取出的数据元素也可能为null。
    * 远程调用返回对象时，一律要求进行空指针判断，防止NPE。
    * 对于Session中获取的数据，建议进行NPE检查，避免空指针。
    * 级联调用obj.getA().getB().getC()；一连串调用，易产生NPE。
    * 正例：使用JDK8的Optional类来防止NPE问题。



11. 【推荐】定义时区分unchecked / checked 异常，避免直接抛出new RuntimeException()，更不允许抛出Exception 或者 Throwable，应使用有业务含义的自定义异常。推荐业界已定义过的自定义异常，如：DAOException / ServiceException等。



12. 【参考】对于公司外的 http/api 开放接口必须使用 errorCode；而应用内部推荐异常抛出；跨应用间 RPC 调用优先考虑使用Result方式，封装 isSuccess() 方法、errorCode、errorMessage；而应用内部直接抛出异常即可。
    * 说明：关于 RPC 方法返回方式使用Result方式的理由：
      * 使用抛异常返回方式，调用方如果没有捕获到就会产生运行时错误。
      * 如果不加栈信息，只是new自定义异常，加入自己的理解的error message，对于调用端解决问题的帮助不会太多。如果加了栈信息，在频繁调用出错的情况下，数据序列化和传输的性能损耗也是问题。



## (二) 日志规约

1. 【强制】应用中不可直接使用日志系统（Log4j、Logback）中的API，而应依赖使用日志框架 （SLF4J、JCL--Jakarta Commons Logging）中的API，使用门面模式的日志框架，有利于维护和各个类的日志处理方式统一。


2. 【强制】所有日志文件至少保存15天，因为有些异常具备以“周”为频次发生的特点。对于当天日志，以“应用名.log”来保存，保存在/`home/admin/应用名/logs/`目录下，过往日志格式为：`{logname}.log.{保存日期}`，日期格式：`yyyy-MM-dd`

  * 正例：以aap应用为例，日志保存在`/home/admin/aapserver/logs/aap.log`，历史日志名称为`aap.log.2016-08-01`

  

3. 【强制】根据国家法律，网络运行状态、网络安全事件、个人敏感信息操作等相关记录，留存的日志不少于六个月，并且进行网络多机备份。



4. 【强制】应用中的扩展日志（如打点、临时监控、访问日志等）命名方式：`appName_logType_logName.log`。logType：日志类型，如stats/monitor/access等；logName：日志描述。这种命名的好处：通过文件名就可知道日志文件属于什么应用，什么类型，什么目的，也有利于归类查找。



5. 【强制】在日志输出时，字符串变量之间的拼接使用占位符的方式。

   * 说明：因为 String 字符串的拼接会使用 StringBuilder 的 append() 方式，有一定的性能损耗。使用占位符仅是替换动作，可以有效提升性能。
   * 正例：`logger.debug("Processing trade with id: {} and symbol: {}", id, symbol);`

   

6. 【强制】对于trace/debug/info级别的日志输出，必须进行日志级别的开关判断。

  * 说明：虽然在 debug(参数) 的方法体内第一行代码 isDisabled(Level.DEBUG_INT) 为真时就直接return，但是参数可能会进行字符串拼接运算。此外，如果debug(getName())这种参数内有getName()方法调用，无谓浪费方法调用的开销。 

  * 正例：

    ```java
    // 如果判断为真，那么可以输出trace和debug级别的日志
    if (logger.isDebugEnabled()) {
        logger.debug("Current ID is: {} and name is: {}", id, getName());
    }
    ```



7. 【强制】避免重复打印日志，浪费磁盘空间，务必在日志配置文件中设置additivity=false。 

   * 正例：`<logger name="com.taobao.dubbo.config" additivity="false">`

   

8. 【强制】生产环境禁止直接使用 System.out 或 System.err 输出日志或使用 e.printStackTrace() 打印异常堆栈。 

   * 说明：标准日志输出与标准错误输出文件每次Jboss重启时才滚动，如果大量输出送往这两个文件，容易造成文件大小超过操作系统大小限制。

   

9. 【强制】异常信息应该包括两类信息：案发现场信息和异常堆栈信息。如果不处理，那么通过关键字 throws往上抛出。

   * 正例：

     ```java
     logger.error("inputParams:{} and errorMessage:{}", 各类参数或者对象toString(), e.getMessage(), e);
     ```

     

10. 【强制】日志打印时禁止直接用JSON工具将对象转换成String。 

    * 说明：如果对象里某些get方法被覆写，存在抛出异常的情况，则可能会因为打印日志而影响正常业务流程的执行。 
    * 正例：打印日志时仅打印出业务相关属性值或者调用其对象的toString()方法。

    

11. 【推荐】谨慎地记录日志。生产环境禁止输出debug日志；有选择地输出info日志；如果使用warn来记录刚上线时的业务行为信息，一定要注意日志输出量的问题，避免把服务器磁盘撑爆，并记得及时删除这些观察日志。



12. 【推荐】可以使用warn日志级别来记录用户输入参数错误的情况，避免用户投诉时，无所适从。如非必要，请不要在此场景打出error级别，避免频繁报警。 
    * 说明：注意日志输出的级别，error级别只记录系统逻辑出错、异常或者重要的错误信息。



13. 【推荐】尽量用英文来描述日志错误信息，如果日志中的错误信息用英文描述不清楚的话使用中文描述即可，否则容易产生歧义。



# 三、单元测试

1. 【强制】单元测试应该是全自动执行的，并且非交互式的。测试用例通常是被定期执行的，执行过程必须完全自动化才有意义。输出结果需要人工检查的测试不是一个好的单元测试。单元测试中不准使用System.out来进行人肉验证，必须使用assert来验证。



2. 【强制】保持单元测试的独立性。为了保证单元测试稳定可靠且便于维护，单元测试用例之间决不能互相调用，也不能依赖执行的先后次序。



3. 【强制】单元测试是可以重复执行的，不能受到外界环境的影响。

   * 说明：单元测试通常会被放到持续集成中，每次有代码check in时单元测试都会被执行。如果单测对外部环境（网络、服务、中间件等）有依赖，容易导致持续集成机制的不可用。
   * 正例：为了不受外界环境影响，要求设计代码时就把SUT的依赖改成注入，在测试时用spring 这样的DI框架注入一个本地（内存）实现或者Mock实现。

   

4. 【强制】对于单元测试，要保证测试粒度足够小，有助于精确定位问题。单测粒度至多是类级别，一般是方法级别。



5. 【强制】单元测试代码必须写在如下工程目录：src/test/java，不允许写在业务代码目录下。 说明：源码编译时会跳过此目录，而单元测试框架默认是扫描此目录。



6. 【推荐】单元测试的基本目标：语句覆盖率达到70%；核心模块的语句覆盖率和分支覆盖率都要达到100% 说明：在工程规约的应用分层中提到的DAO层，Manager层，可重用度高的Service，都应该进行单元测试。



7. 【推荐】对于数据库相关的查询，更新，删除等操作，不能假设数据库里的数据是存在的，或者直接操作数据库把数据插入进去，请使用程序插入或者导入数据的方式来准备数据。 

   * 反例：删除某一行数据的单元测试，在数据库中，先直接手动增加一行作为删除目标，但是这一行新增数据并不符合业务插入规则，导致测试结果异常。 

   

8. 【推荐】和数据库相关的单元测试，可以设定自动回滚机制，不给数据库造成脏数据。或者对单元测试产生的数据有明确的前后缀标识。 
   * 正例：在阿里巴巴企业智能事业部的内部单元测试中，使用ENTERPRISE_INTELLIGENCE _UNIT_TEST_的前缀来标识单元测试相关代码。



9. 【推荐】对于不可测的代码在适当的时机做必要的重构，使代码变得可测，避免为了达到测试要求而书写不规范测试代码。



10. 【推荐】在设计评审阶段，开发人员需要和测试人员一起确定单元测试范围，单元测试最好覆盖所有测试用例（UC）。 



11. 【推荐】单元测试作为一种质量保障手段，在项目提测前完成单元测试，不建议项目发布后补充单元测试用例。



12. 【参考】为了更方便地进行单元测试，业务代码应避免以下情况：
    * 构造方法中做的事情过多。
    * 存在过多的全局变量和静态方法。
    * 存在过多的外部依赖。
    * 存在过多的条件语句。 
    * 说明：多层条件语句建议使用卫语句、策略模式、状态模式等方式重构。



# 四、安全规约

1. 【强制】隶属于用户个人的页面或者功能必须进行权限控制校验。

   * 说明：防止没有做水平权限校验就可随意访问、修改、删除别人的数据，比如查看他人的私信内容。

   

2. 【强制】用户敏感数据禁止直接展示，必须对展示数据进行脱敏。

   * 说明：中国大陆个人手机号码显示：1391219，隐藏中间4位，防止隐私泄露。

   

3. 【强制】用户输入的SQL参数严格使用参数绑定或者METADATA字段值限定，防止SQL注入，禁止字符串拼接SQL访问数据库。

  * 反例：某系统签名大量被恶意修改，即是因为对于危险字符 # --没有进行转义，导致数据库更新时，where后边的信息被注释掉，对全库进行更新。

  

4. 【强制】用户请求传入的任何参数必须做有效性验证。

  * 说明：忽略参数校验可能导致：
    * page size过大导致内存溢出
    * 恶意order by导致数据库慢查询
    * 缓存击穿
    * SSRF
    * 任意重定向
    * SQL注入，Shell注入，反序列化注入
    * 正则输入源串拒绝服务ReDoS
  * Java代码用正则来验证客户端的输入，有些正则写法验证普通用户输入没有问题，但是如果攻击人员使用的是特殊构造的字符串来验证，有可能导致死循环的结果。



5. 【强制】禁止向HTML页面输出未经安全过滤或未正确转义的用户数据。



6. 【强制】表单、AJAX提交必须执行CSRF安全验证。
   * 说明：CSRF(Cross-site request forgery)跨站请求伪造是一类常见编程漏洞。对于存在CSRF漏洞的应用/网站，攻击者可以事先构造好URL，只要受害者用户一访问，后台便在用户不知情的情况下对数据库中用户参数进行相应修改。



7. 【强制】URL外部重定向传入的目标地址必须执行白名单过滤。



8. 【强制】在使用平台资源，譬如短信、邮件、电话、下单、支付，必须实现正确的防重放的机制，如数量限制、疲劳度控制、验证码校验，避免被滥刷而导致资损。 说明：如注册时发送验证码到手机，如果没有限制次数和频率，那么可以利用此功能骚扰到其它用户，并造成短信平台资源浪费。



9. 【推荐】发贴、评论、发送即时消息等用户生成内容的场景必须实现防刷、文本内容违禁词过滤等风控策略。



# 五、MySQL数据库
## (一) 建表规约

1. 【强制】表达是与否概念的字段，必须使用 is_xxx 的方式命名，数据类型是 unsigned tinyint（1表示是，0表示否）。
   * 说明：任何字段如果为非负数，必须是unsigned。 注意：POJO类中的任何布尔类型的变量，都不要加is前缀，所以，需要在 `<resultMap>` 设置从 is_xxx 到 Xxx 的映射关系。数据库表示是与否的值，使用tinyint 类型，坚持 is_xxx 的命名方式是为了明确其取值含义与取值范围。
   * 正例：表达逻辑删除的字段名is_deleted，1表示删除，0表示未删除。



2. 【强制】表名、字段名必须使用小写字母或数字，禁止出现数字开头，禁止两个下划线中间只出现数字。

   * 说明：MySQL在Windows下不区分大小写，但在Linux下默认是区分大小写。因此，数据库名、表名、字段名，都不允许出现任何大写字母，避免节外生枝。
   * 正例：aliyun_admin，rdc_config，level3_name
   * 反例：AliyunAdmin，rdcConfig，level_3_name

   

3. 【强制】表名不使用复数名词。



4. 【强制】禁用保留字，如desc、range、match、delayed等，请参考MySQL官方保留字。



5. 【强制】主键索引名为`pk_字段名`，唯一索引名为`uk_字段名`，普通索引名则为`idx_字段名`。

   * 说明：pk_  即 primary key；uk_  即 unique key；idx_  即 index 的简称。

   

6. 【强制】小数类型为decimal，禁止使用float和double。

   * 说明：在存储的时候，float 和 double 都存在精度损失的问题，很可能在比较值的时候，得到不正确的结果。如果存储的数据范围超过 decimal 的范围，建议将数据拆成整数和小数并分开存储。

   

7. 【强制】如果存储的字符串长度几乎相等，使用char定长字符串类型。



8. 【强制】varchar是可变长字符串，不预先分配存储空间，长度不要超过5000，如果存储长度大于此值，定义字段类型为text，独立出来一张表，用主键来对应，避免影响其它字段索引效率。



9. 【强制】表必备三字段：id, create_time, update_time。
   * 说明：其中id必为主键，类型为bigint unsigned、单表时自增、步长为1。create_time, update_time的类型均为 datetime 类型，前者现在时表示主动式创建，后者过去分词表示被动式更新。



10. 【推荐】表的命名最好是遵循“业务名称_表的作用”。



11. 【推荐】库名与应用名称尽量一致。



12. 【推荐】如果修改字段含义或对字段表示的状态追加时，需要及时更新字段注释。



13. 【推荐】字段允许适当冗余，以提高查询性能，但必须考虑数据一致。冗余字段应遵循：

    * 不是频繁修改的字段。
    * 不是唯一索引的字段。
    * 不是varchar超长字段，更不能是text字段。

    

14. 【推荐】单表行数超过500万行或者单表容量超过2GB，才推荐进行分库分表。 
    * 说明：如果预计三年后的数据量根本达不到这个级别，请不要在创建表时就分库分表。



## (二) 索引规约

1. 【强制】业务上具有唯一特性的字段，即使是组合字段，也必须建成唯一索引。

   * 说明：不要以为唯一索引影响了 insert 速度，这个速度损耗可以忽略，但提高查找速度是明显的；另外，即使在应用层做了非常完善的校验控制，只要没有唯一索引，根据墨菲定律，必然有脏数据产生。

   

2. 【强制】超过三个表禁止 join。需要 join 的字段，数据类型保持绝对一致；多表关联查询时，保证被关联的字段需要有索引。



3. 【强制】在 varchar 字段上建立索引时，必须指定索引长度，没必要对全字段建立索引，根据实际文本区分度决定索引长度。

   * 说明：索引的长度与区分度是一对矛盾体，一般对字符串类型数据，长度为20的索引，区分度会高达90%以上，可以使用 `count(distinct left(列名, 索引长度))/count(*)` 的区分度来确定。

   

4. 【强制】页面搜索严禁左模糊或者全模糊，如果需要请走搜索引擎来解决。

   * 说明：索引文件具有B-Tree的最左前缀匹配特性，如果左边的值未确定，那么无法使用此索引。

   

5. 【推荐】如果有 order by 的场景，请注意利用索引的有序性。order by 最后的字段是组合索引的一部分，并且放在索引组合顺序的最后，避免出现file_sort的情况，影响查询性能。

   * 正例：where a=? and b=? order by c; 索引：a_b_c
   * 反例：索引如果存在范围查询，那么索引有序性无法利用，如：WHERE a>10 ORDER BY b; 索引a_b无法排序。

   

6. 【推荐】利用覆盖索引来进行查询操作，避免回表。

   * 正例：能够建立索引的种类分为主键索引、唯一索引、普通索引三种，而覆盖索引只是一种查询的一种效果，用explain的结果，extra列会出现：using index。

   

7. 【推荐】利用延迟关联或者子查询优化超多分页场景。

   * 说明：MySQL并不是跳过offset行，而是取 offset+N 行，然后返回放弃前offset行，返回N行，那当offset特别大的时候，效率就非常的低下，要么控制返回的总页数，要么对超过特定阈值的页数进行SQL改写。

   * 正例：先快速定位需要获取的id段，然后再关联： 

     `SELECT t1.* FROM 表1 as t1, (select id from 表1 where 条件 LIMIT 100000,20 ) as t2 where t1.id = t2.id`

   

8. 【推荐】SQL性能优化的目标：至少要达到 range 级别，要求是ref级别，如果可以是 consts 最好。

   * consts 单表中最多只有一个匹配行（主键或者唯一索引），在优化阶段即可读取到数据。
   * ref 指的是使用普通的索引（normal index）。
   * range 对索引进行范围检索。
   * 反例：explain表的结果，type=index，索引物理文件全扫描，速度非常慢，这个index级别比较range还低，与全表扫描是小巫见大巫。

   

9. 【推荐】建组合索引的时候，区分度最高的在最左边。 正例：如果where a=? and b=?，a列的几乎接近于唯一值，那么只需要单建idx_a索引即可。 

   * 说明：存在非等号和等号混合判断条件时，在建索引时，请把等号条件的列前置。如：where c>? and d=? 那么即使c的区分度更高，也必须把d放在索引的最前列，即建立组合索引idx_d_c。

   

10. 【推荐】防止因字段类型不同造成的隐式转换，导致索引失效。



11. 【参考】创建索引时避免有如下极端误解：
    * 索引宁滥勿缺。认为一个查询就需要建一个索引。
    * 吝啬索引的创建。认为索引会消耗空间、严重拖慢记录的更新以及行的新增速度。
    * 抵制惟一索引。认为惟一索引一律需要在应用层通过“先查后插”方式解决。



## (三) SQL语句

1. 【强制】不要使用 `count(列名)` 或 `count(常量)` 来替代 `count(*)`，`count(*)` 是 SQL92 定义的标准统计行数的语法，跟数据库无关，跟 NULL 和非 NULL 无关。
   * 说明：`count(*)` 会统计值为 NULL 的行，而 `count(列名)` 不会统计此列为NULL值的行。



2. 【强制】`count(distinct col)` 计算该列除 NULL 之外的不重复行数，注意 `count(distinct col1, col2)` 如果其中一列全为 NULL，那么即使另一列有不同的值，也返回为0。



3. 【强制】当某一列的值全是 NULL 时，`count(col)` 的返回结果为0，但 `sum(col)` 的返回结果为 NULL，因此使用 `sum()` 时需注意 NPE 问题。
   * 正例：可以使用如下方式来避免 sum 的 NPE 问题：`SELECT IFNULL(SUM(column), 0) FROM table;`



4. 【强制】使用 `ISNULL()` 来判断是否为 NULL 值。

   * 说明：NULL与任何值的直接比较都为NULL。
     * NULL <> NULL 的返回结果是 NULL，而不是false。
     * NULL = NULL 的返回结果是 NULL，而不是true。
     * NULL <> 1 的返回结果是 NULL，而不是true。
   * 反例：在SQL语句中，如果在null前换行，影响可读性。`select * from table where column1 is null and column3 is not null;` 而 `ISNULL(column)`是一个整体，简洁易懂。从性能数据上分析，`ISNULL(column)`执行效率更快一些。

   

5. 【强制】代码中写分页查询逻辑时，若count为0应直接返回，避免执行后面的分页语句。



6. 【强制】不得使用外键与级联，一切外键概念必须在应用层解决。
   * 说明：（概念解释）学生表中的student_id是主键，那么成绩表中的student_id则为外键。如果更新学生表中的student_id，同时触发成绩表中的student_id更新，即为级联更新。外键与级联更新适用于单机低并发，不适合分布式、高并发集群；级联更新是强阻塞，存在数据库更新风暴的风险；外键影响数据库的插入速度。



7. 【强制】禁止使用存储过程，存储过程难以调试和扩展，更没有移植性。



8. 【强制】数据订正（特别是删除或修改记录操作）时，要先select，避免出现误删除，确认无误才能执行更新语句。



9. 【强制】对于数据库中表记录的查询和变更，只要涉及多个表，都需要在列名前加表的别名（或表名）进行限定。

   * 说明：对多表进行查询记录、更新记录、删除记录时，如果对操作列没有限定表的别名（或表名），并且操作列在多个表中存在时，就会抛异常。
   * 正例：`select t1.name from table_first as t1 , table_second as t2 where t1.id=t2.id;` 
   * 反例：在某业务中，由于多表关联查询语句没有加表的别名（或表名）的限制，正常运行两年后，最近在某个表中增加一个同名字段，在预发布环境做数据库变更后，线上查询语句出现出1052异常：Column 'name' in field list is ambiguous。

   

10. 【推荐】SQL语句中表的别名前加as，并且以t1、t2、t3、...的顺序依次命名。

    * 说明：
      * 别名可以是表的简称，或者是依照表在SQL语句中出现的顺序，以t1、t2、t3的方式命名。
      * 别名前加 as 使别名更容易识别。
    * 正例：`select t1.name from table_first as t1, table_second as t2 where t1.id=t2.id;`



11. 【推荐】in操作能避免则避免，若实在避免不了，需要仔细评估in后边的集合元素数量，控制在1000个之内。



12. 【参考】因国际化需要，所有的字符存储与表示，均采用 utf8 字符集，那么字符计数方法需要注意。

    * 说明： 
      * SELECT LENGTH("轻松工作")； 返回为12
      * SELECT CHARACTER_LENGTH("轻松工作")； 返回为4
      * 如果需要存储表情，那么选择 utf8mb4 来进行存储，注意它与 utf8 编码的区别。

    

13. 【参考】TRUNCATE TABLE 比 DELETE 速度快，且使用的系统和事务日志资源少，但TRUNCATE无事务且不触发trigger，有可能造成事故，故不建议在开发代码中使用此语句。

    * 说明：TRUNCATE TABLE 在功能上与不带 WHERE 子句的 DELETE 语句相同。



## (四) ORM映射

1. 【强制】在表查询中，一律不要使用 * 作为查询的字段列表，需要哪些字段必须明确写明。

   * 说明：

     * 增加查询分析器解析成本。
     * 增减字段容易与resultMap配置不一致。
     * 无用字段增加网络消耗，尤其是text类型的字段。

     

2. 【强制】POJO类的布尔属性不能加is，而数据库字段必须加is_，要求在resultMap中进行字段与属性之间的映射。

   * 说明：参见定义POJO类以及数据库字段定义规定，在 sql.xml 增加映射，是必须的。

   

3. 【强制】不要用 resultClass 当返回参数，即使所有类属性名与数据库字段一一对应，也需要定义`<resultMap>`；反过来，每一个表也必然有一个 `<resultMap>` 与之对应。

   * 说明：配置映射关系，使字段与DO类解耦，方便维护。

   

4. 【强制】sql.xml配置参数使用：#{}，#param# 不要使用${} 此种方式容易出现SQL注入。



5. 【强制】iBATIS 自带的 `queryForList(String statementName,int start,int size)` 不推荐使用。

   * 说明：其实现方式是在数据库取到 statementName 对应的SQL语句的所有记录，再通过 subList 取start,size 的子集合。
     正例：

     ```java
     Map<String, Object> map = new HashMap<>(16);
     map.put("start", start);
     map.put("size", size);
     ```

     

6. 【强制】不允许直接拿 HashMap 与 Hashtable 作为查询结果集的输出。

   * 反例：某同学为避免写一个 `<resultMap>xxx</resultMap>`，直接使用 HashTable 来接收数据库返回结果，结果出现日常是把 bigint 转成 Long 值，而线上由于数据库版本不一样，解析成 BigInteger，导致线上问题。

   

7. 【强制】更新数据表记录时，必须同时更新记录对应的update_time字段值为当前时间。

   

8. 【推荐】不要写一个大而全的数据更新接口。传入为POJO类，不管是不是自己的目标更新字段，都进行`update table set c1=value1,c2=value2,c3=value3; ` 这是不对的。执行SQL时，不要更新无改动的字段，一是易出错；二是效率低；三是增加binlog存储。



9. 【参考】@Transactional事务不要滥用。事务会影响数据库的 QPS，另外使用事务的地方需要考虑各方面的回滚方案，包括缓存回滚、搜索引擎回滚、消息补偿、统计修正等。



10. 【参考】`<isEqual>` 中的 compareValue 是与属性值对比的常量，一般是数字，表示相等时带上此条件；`<isNotEmpty>` 表示不为空且不为null时执行；`<isNotNull>` 表示不为null值时执行。



# 六、工程结构
## (一) 应用分层

1. 【推荐】根据业务架构实践，结合业界分层规范与流行技术框架分析，推荐分层结构如图所示，默认上层依赖于下层，箭头关系表示可直接依赖，如：开放API层可以依赖于Web层（Controller层），也可以直接依赖于Service层，依此类推：

![应用分层](https://raw.githubusercontent.com/Cavielee/notePics/main/应用分层.jpg)

* 开放API层：可直接封装Service接口暴露成RPC接口；通过Web封装成http接口；网关控制层等。
* 终端显示层：各个端的模板渲染并执行显示的层。当前主要是velocity渲染，JS渲染，JSP渲染，移动端展示等。
* Web层：主要是对访问控制进行转发，各类基本参数校验，或者不复用的业务简单处理等。
* Service层：相对具体的业务逻辑服务层。
* Manager层：通用业务处理层，它有如下特征： 1） 对第三方平台封装的层，预处理返回结果及转化异常信息，适配上层接口。 2） 对Service层通用能力的下沉，如缓存方案、中间件通用处理。 3） 与DAO层交互，对多个DAO的组合复用。
* DAO层：数据访问层，与底层MySQL、Oracle、Hbase、OB等进行数据交互。
  • 第三方服务：包括其它部门RPC服务接口，基础平台，其它公司的HTTP接口，如淘宝开放平台、支付宝付款服务、高德地图服务等。
* 外部数据接口：外部（应用）数据存储服务提供的接口，多见于数据迁移场景中。



2. 【参考】（分层异常处理规约）在DAO层，产生的异常类型有很多，无法用细粒度的异常进行catch，使用catch(Exception e)方式，并throw new DAOException(e)，不需要打印日志，因为日志在Manager/Service层一定需要捕获并打印到日志文件中去，如果同台服务器再打日志，浪费性能和存储。在Service层出现异常时，必须记录出错日志到磁盘，尽可能带上参数信息，相当于保护案发现场。Manager层与Service同机部署，日志方式与DAO层处理一致，如果是单独部署，则采用与Service一致的处理方式。Web层绝不应该继续往上抛异常，因为已经处于顶层，如果意识到这个异常将导致页面无法正常渲染，那么就应该直接跳转到友好错误页面，尽量加上友好的错误提示信息。开放接口层要将异常处理成错误码和错误信息方式返回。



3. 【参考】分层领域模型规约：
   * DO（Data Object）：此对象与数据库表结构一一对应，通过DAO层向上传输数据源对象。
   * DTO（Data Transfer Object）：数据传输对象，Service或Manager向外传输的对象。
   * BO（Business Object）：业务对象，可以由Service层输出的封装业务逻辑的对象。
   * Query：数据查询对象，各层接收上层的查询请求。注意超过2个参数的查询封装，禁止使用Map类来传输。
   * VO（View Object）：显示层对象，通常是Web向模板渲染引擎层传输的对象。



## (二) 二方库依赖

1. 【强制】定义GAV遵从以下规则：

   * GroupID格式：`com.{公司/BU }.业务线 [.子业务线]`，最多4级。
     * 说明：{公司/BU} 例如：alibaba/taobao/tmall/aliexpress等BU一级；子业务线可选。
     * 正例：com.taobao.jstorm 或 com.alibaba.dubbo.register
   * ArtifactID格式：产品线名-模块名。语义不重复不遗漏，先到中央仓库去查证一下。
     * 正例：dubbo-client / fastjson-api / jstorm-tool 3）
   * Version：详细规定参考下方。

   

2. 【强制】二方库版本号命名方式：`主版本号.次版本号.修订号`

  * 主版本号：产品方向改变，或者大规模API不兼容，或者架构不兼容升级。
  * 次版本号：保持相对兼容性，增加主要功能特性，影响范围极小的API不兼容修改。
  * 修订号：保持完全兼容性，修复BUG、新增次要功能特性等。 说明：注意起始版本号必须为：1.0.0，而不是0.0.1。
  * 反例：仓库内某二方库版本号从1.0.0.0开始，一直默默“升级”成1.0.0.64，完全失去版本的语义信息。

  

3. 【强制】线上应用不要依赖SNAPSHOT版本（安全包除外）；正式发布的类库必须先去中央仓库进行查证，使RELEASE版本号有延续性，且版本号不允许覆盖升级。

   * 说明：不依赖SNAPSHOT版本是保证应用发布的幂等性。另外，也可以加快编译时的打包构建。 

   

4. 【强制】二方库的新增或升级，保持除功能点之外的其它jar包仲裁结果不变。如果有改变，必须明确评估和验证。

   * 说明：在升级时，进行 `dependency:resolve` 前后信息比对，如果仲裁结果完全不一致，那么通过`dependency:tree` 命令，找出差异点，进行 `<exclude>` 排除jar包。

   

5. 【强制】二方库里可以定义枚举类型，参数可以使用枚举类型，但是接口返回值不允许使用枚举类型或者包含枚举类型的POJO对象。



6. 【强制】依赖于一个二方库群时，必须定义一个统一的版本变量，避免版本号不一致。
   * 说明：依赖 springframework-core,-context,-beans，它们都是同一个版本，可以定义一个变量来保存版本：${spring.version}，定义依赖的时候，引用该版本。



7. 【强制】禁止在子项目的pom依赖中出现相同的GroupId，相同的ArtifactId，但是不同的Version。
   * 说明：在本地调试时会使用各子项目指定的版本号，但是合并成一个war，只能有一个版本号出现在最后的lib目录中。曾经出现过线下调试是正确的，发布到线上却出故障的先例。



8. 【推荐】底层基础技术框架、核心数据管理平台、或近硬件端系统谨慎引入第三方实现。



9. 【推荐】所有pom文件中的依赖声明放在 `<dependencies>` 语句块中，所有版本仲裁放在`<dependencyManagement>` 语句块中。

   * 说明：`<dependencyManagement>` 里只是声明版本，并不实现引入，因此子项目需要显式的声明依赖，version和scope都读取自父 pom。而 `<dependencies>` 所有声明在主pom的 `<dependencies>` 里的依赖都会自动引入，并默认被所有的子项目继承。

   

10. 【推荐】二方库不要有配置项，最低限度不要再增加配置项。



11. 【推荐】不要使用不稳定的工具包或者Utils类。

    * 说明：不稳定指的是提供方无法做到向下兼容，在编译阶段正常，但在运行时产生异常，因此，尽量使用业界稳定的二方工具包。

    

12. 【参考】为避免应用二方库的依赖冲突问题，二方库发布者应当遵循以下原则：
    * 精简可控原则。移除一切不必要的API和依赖，只包含 Service API、必要的领域模型对象、Utils类、常量、枚举等。如果依赖其它二方库，尽量是provided引入，让二方库使用者去依赖具体版本号；无log具体实现，只依赖日志框架。
    * 稳定可追溯原则。每个版本的变化应该被记录，二方库由谁维护，源码在哪里，都需要能方便查到。除非用户主动升级版本，否则公共二方库的行为不应该发生变化。



## (三) 服务器

1. 【推荐】高并发服务器建议调小 TCP 协议的 time_wait 超时时间。

   * 说明：操作系统默认240秒后，才会关闭处于 time_wait 状态的连接，在高并发访问下，服务器端会因为处于 time_wait 的连接数太多，可能无法建立新的连接，所以需要在服务器上调小此等待值。
   * 正例：在 linux 服务器上请通过变更 `/etc/sysctl.conf` 文件去修改该缺省值（秒）： `net.ipv4.tcp_fin_timeout = 30`

   

2. 【推荐】调大服务器所支持的最大文件句柄数（File Descriptor，简写为fd）。

   * 说明：主流操作系统的设计是将 TCP/UDP 连接采用与文件一样的方式去管理，即一个连接对应于一个fd。主流的 linux 服务器默认所支持最大 fd 数量为 1024，当并发连接数很大时很容易因为 fd 不足而出现“open too many files”错误，导致新的连接无法建立。建议将linux服务器所支持的最大句柄数调高数倍（与服务器的内存数量相关）。

     

3. 【推荐】给 JVM 环境参数设置 `-XX:+HeapDumpOnOutOfMemoryError` 参数，让 JVM 碰到 OOM 场景时输出dump信息。

   * 说明：OOM的发生是有概率的，甚至相隔数月才出现一例，出错时的堆内信息对解决问题非常有帮助。



4. 【推荐】在线上生产环境，JVM 的 Xms 和 Xmx 设置一样大小的内存容量，避免在GC 后调整堆大小带来的压力。



5. 【参考】服务器内部重定向必须使用forward；外部重定向地址必须使用 URL Broker 生成，否则因线上采用HTTPS 协议而导致浏览器提示“不安全“。此外，还会带来URL维护不一致的问题。



# 七、设计规约 

1. 【强制】存储方案和底层数据结构的设计获得评审一致通过，并沉淀成为文档。

   * 说明：有缺陷的底层数据结构容易导致系统风险上升，可扩展性下降，重构成本也会因历史数据迁移和系统平滑过渡而陡然增加，所以，存储方案和数据结构需要认真地进行设计和评审，生产环境提交执行后，需要进行double check。
   * 正例：评审内容包括存储介质选型、表结构设计能否满足技术方案、存取性能和存储空间能否满足业务发展、表或字段之间的辩证关系、字段名称、字段类型、索引等；数据结构变更（如在原有表中新增字段）也需要进行评审通过后上线。

   

2. 【强制】在需求分析阶段，如果与系统交互的User超过一类并且相关的User Case超过5个，使用用例图来表达更加清晰的结构化需求。

   

3. 【强制】如果某个业务对象的状态超过3个，使用状态图来表达并且明确状态变化的各个触发条件。 说明：状态图的核心是对象状态，首先明确对象有多少种状态，然后明确两两状态之间是否存在直接转换关系，再明确触发状态转换的条件是什么。 正例：淘宝订单状态有已下单、待付款、已付款、待发货、已发货、已收货等。比如已下单与已收货这两种状态之间是不可能有直接转换关系的。



4. 【强制】如果系统中某个功能的调用链路上的涉及对象超过3个，使用时序图来表达并且明确各调用环节的输入与输出。
   * 说明：时序图反映了一系列对象间的交互与协作关系，清晰立体地反映系统的调用纵深链路。



5. 【强制】如果系统中模型类超过5个，并且存在复杂的依赖关系，使用类图来表达并且明确类之间的关系。

   * 说明：类图像建筑领域的施工图，如果搭平房，可能不需要，但如果建造蚂蚁Z空间大楼，肯定需要详细的施工图。

   

6. 【强制】如果系统中超过2个对象之间存在协作关系，并且需要表示复杂的处理流程，使用活动图来表示。

   * 说明：活动图是流程图的扩展，增加了能够体现协作关系的对象泳道，支持表示并发等。



7. 【推荐】系统架构设计时明确以下目标：
   * 确定系统边界。确定系统在技术层面上的做与不做。
   * 确定系统内模块之间的关系。确定模块之间的依赖关系及模块的宏观输入与输出。
   * 确定指导后续设计与演化的原则。使后续的子系统或模块设计在一个既定的框架内和技术方向上继续演化。
