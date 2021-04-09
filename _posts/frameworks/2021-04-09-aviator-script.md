---
layout: post
title: Aviator规则引擎
categories: 脚本语言
excerpt: 单例模式要求类能够有返回对象一个引用(永远是同一个)和一个获得该实例的方法（必须是静态方法，通常使用getInstance这个名称）。
tags: Aviator
---

### 一、 基本特性
1. 支持数字、字符串、正则表达式、布尔值、正则表达式等基本类型，完整支持所有 Java 运算符及优先级等。
2. 函数是一等公民，支持闭包和函数式编程。
3. 内置 bigint/decmal 类型用于大整数和高精度运算，支持运算符重载得以让这些类型使用普通的算术运算符 +-*/ 参与运算。
4. 完整的脚本语法支持，包括多行数据、条件语句、循环语句、词法作用域和异常处理等。
5. 函数式编程结合 Sequence 抽象，便捷处理任何集合。
6. 轻量化的模块系统。
7. 多种方式，方便地调用 Java 方法，完整支持 Java 脚本 API（方便从 Java 调用脚本）。
8. 丰富的定制选项，可作为安全的语言沙箱和全功能语言使用。
9. 轻量化，高性能，通过直接将脚本翻译成 JVM 字节码，AviatorScript 的基础性能较好。


#### 使用场景包括：

1. 规则判断及规则引擎
2. 公式计算
3. 动态脚本控制
4. 集合数据 ELT 等 ……


* 官方文档: <https://www.yuque.com/boyan-avfmj/aviatorscript>


### 二、 项目实战

#### 1. 创建maven工程

添加pom.xml依赖文件

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <groupId>vip.sunjin</groupId>
    <artifactId>HelloAviator</artifactId>
    <version>1.0-SNAPSHOT</version>

    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
        <java.version>1.8</java.version>
        <grpc.version>1.36.1</grpc.version>
        <protobuf.version>3.15.6</protobuf.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>com.googlecode.aviator</groupId>
            <artifactId>aviator</artifactId>
            <version>5.2.4</version>
        </dependency>
        <dependency>
            <groupId>junit</groupId>
            <artifactId>junit</artifactId>
            <version>4.13.2</version>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>com.alibaba</groupId>
            <artifactId>fastjson</artifactId>
            <version>1.2.76</version>
        </dependency>
        <dependency>
            <groupId>org.apache.commons</groupId>
            <artifactId>commons-lang3</artifactId>
            <version>3.12.0</version>
        </dependency>
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <version>1.18.20</version>
            <scope>provided</scope>
        </dependency>
    </dependencies>


    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
</project>
```


#### 2. 开始测试

写一个JUNIT测试类，就可以开始测试了

* 可以直接调用AviatorEvaluator.execute执行脚本

```java
public class ExpressionUT {

    @Before
    public void before(){
        AviatorEvaluator.setOption(Options.ALWAYS_PARSE_FLOATING_POINT_NUMBER_INTO_DECIMAL, true);
    }
     /**
     * 数学运算
     */
    @Test
    public void testMath() {
        Long result = (Long) AviatorEvaluator.execute("(1 + 2 + 3) * 4 - (10 / 2)");
        System.out.println(result);// result = 19
    
        Object result2 = AviatorEvaluator.execute("10 % 3");
        System.out.println(result2);// result = 19
    }
    
    
    /**
     * 逻辑运算和比较运算, 三元运算
     */
    @Test
    public void testLogic() {
        Boolean result = (Boolean) AviatorEvaluator.execute("10 > 5 && 9 < 18 || 100 != 200");
        System.out.println(result);// result = true
    
        Map<String, Object> env = new HashMap<>();
        env.put("a", 100.3);
        String result2 = (String) AviatorEvaluator.execute("a>0? 'yes':'no'", env);
        System.out.println(result2);// result2 = yes
    }
}
```

* 也可以先编译，在执行。

```java
 /**
 * 参数绑定
 */
@Test
public void testParam() {
    String expression = "a-(b-c)>100";
    Expression compiledExp = AviatorEvaluator.compile(expression);
    Map<String, Object> env = new HashMap<>();
    env.put("a", 100.3);
    env.put("b", 45);
    env.put("c", -199.100);
    Boolean result = (Boolean) compiledExp.execute(env);
    System.out.println(result); // result = false

}
```

* 除了以上这些，还支持各种运算方式

```java
/**
 * 正则表达式：  通过=~操作符
 */
@Test
public void testRegular() {

    String email = "killme2008@gmail.com";
    Map<String, Object> env = new HashMap<>();
    env.put("email", email);
    Boolean result = (Boolean) AviatorEvaluator.execute("email=~/([\\w0-8]+@\\w+[\\.\\w+]+)/", env);
    System.out.println(result);// result = true
}


/**
 * 内置函数: 自然对数
 */
@Test
public void testFunction() {
    String expression = "math.log(a)";
    Expression compiledExp = AviatorEvaluator.compile(expression);
    Map<String, Object> env = new HashMap<>();
    env.put("a", 100.3);

    Double result = (Double) compiledExp.execute(env);
    System.out.println(result); // result = 4.60816569496789

}
```

* 支持自定义函数

求和
```java
public class MathSumFunction extends AbstractVariadicFunction {
    @Override
    public AviatorObject variadicCall(Map<String, Object> env, AviatorObject... args) {
        if (args.length == 0) {
            return AviatorRuntimeJavaType.valueOf(BigDecimal.ZERO);
        }
        BigDecimal sum = BigDecimal.ZERO;
        //calculate total
        for (AviatorObject arg : args) {
            Object param = arg.getValue(env);
            sum = sum.add(new BigDecimal(param.toString()));
        }

        return AviatorRuntimeJavaType.valueOf(sum);
    }

    @Override
    public String getName() {
        return "math.sum";
    }
}
```

测试
```java
 /**
 * 自定义函数
 */
@Test
public void testSum() {
    AviatorEvaluator.addFunction(new MathSumFunction());

    Expression compiledExp = AviatorEvaluator.compile("math.sum()", true);
    BigDecimal result =  (BigDecimal)compiledExp.execute(env);
    Assert.assertEquals(0, result.intValue());
    System.out.println(result);

    Expression compiledExp2 = AviatorEvaluator.compile("math.sum(a,b,c,d)", true);
    BigDecimal result2 =  (BigDecimal)compiledExp2.execute(env);
    Assert.assertEquals(170.8, result2.doubleValue(),0);
    System.out.println(result2);
}
```



* 支持直接调用Java类库中的方法

自定义静态方法
```java
public class MathFunctions {

    private MathFunctions() throws IllegalAccessException {
        throw new IllegalAccessException();
    }


    public static double exp(double a){
        return Math.exp(a);
    }

    public static double mod(double a, double b){
        return a % b;
    }


    public static boolean isBlank(Object var){
        if(var == null){
            return true;
        }
        return StringUtils.isBlank(var.toString());
    }

    public static boolean in(Object var, String params){
        if(var == null || params == null){
            return false;
        }
        String[] arrays = params.split(",");
        String str = var.toString();
        for(String param: arrays){
            if(param.equals(str)){
                return true;
            }
        }
        return false;
    }


    public static double avg(List<Double> list){
        if(list == null){
            return 0;
        }
        return list.stream().mapToDouble(Double::doubleValue).average().orElse(0);
    }


    public static double sum(List<Double> list){
        if(list == null){
            return 0;
        }
        return list.stream().mapToDouble(Double::doubleValue).sum();
    }

    @Function(rename = "FLOOR")
    public static double floor(double a, int n){
        BigDecimal decimal = BigDecimal.valueOf(a);

        return decimal.setScale(n,BigDecimal.ROUND_FLOOR).doubleValue();
    }

    public static double CEIL(double a, int n){
        BigDecimal decimal = BigDecimal.valueOf(a);

        return decimal.setScale(n,BigDecimal.ROUND_CEILING).doubleValue();
    }

    public static double ROUND(double a, int n){
        BigDecimal decimal = BigDecimal.valueOf(a);

        return decimal.setScale(n,BigDecimal.ROUND_HALF_UP).doubleValue();
    }

}
```


```java
@Test
public void testLogical() throws NoSuchMethodException, IllegalAccessException {
    AviatorEvaluator.addStaticFunctions("math", MathFunctions.class);
    Map<String, Object> env = new HashMap<>();
    env.put("a", "   ");
    env.put("b", 12.88);
    env.put("c", null);
    env.put("d", 20.4);
    List<Double> list = new ArrayList<>();
    list.add(100.3);
    list.add(45.5);
    list.add(20.4);
    env.put("list", list);

    Boolean result = (Boolean) AviatorEvaluator.execute("math.isBlank(a)", env);
    System.out.println(result);// true

    Boolean result2 = (Boolean) AviatorEvaluator.execute("math.isBlank(b)", env);
    System.out.println(result2);// false

    Boolean result3 = (Boolean) AviatorEvaluator.execute("math.isBlank(c)", env);
    System.out.println(result3);// true

    Boolean result4 = (Boolean) AviatorEvaluator.execute("math.in(d, '100.3, 45.5, 20.4')", env);
    System.out.println(result4);// true

    Boolean result5 = (Boolean) AviatorEvaluator.execute("math.in(b, '100.3, 45.5, 20.4')", env);
    System.out.println(result5);// false
}

@Test
public void testJavaMethod() throws NoSuchMethodException, IllegalAccessException {
    AviatorEvaluator.addStaticFunctions("StringUtils", StringUtils.class); //org.apache.commons.lang3.StringUtils;
    AviatorEvaluator.addStaticFunctions("math", Math.class);
    AviatorEvaluator.addStaticFunctions("math", MathFunctions.class);

    Map<String, Object> env = new HashMap<>();
    env.put("a", "   ");
    env.put("b", 12.123);
    List<Double> list = new ArrayList<>();
    list.add(100.3);
    list.add(45.5);
    list.add(20.4);
    env.put("list", list);

    Boolean result = (Boolean) AviatorEvaluator.execute("StringUtils.isBlank(a)", env);
    System.out.println(result);// true

    Double result2 = (Double) AviatorEvaluator.execute("math.floor(b)", env);
    System.out.println(result2);// 12.0

    Double result4 = (Double) AviatorEvaluator.execute("math.avg(list)", env);
    System.out.println(result4);// 55.4

    Object result5 = AviatorEvaluator.execute("math.sum(list)", env);
    System.out.println(result5);// 166.2
}
```


* 其他自定义函数示例

计算方差
```java
public class MathVarianceFunction extends AbstractVariadicFunction {
    private static final int PRECISION = 14;

    @Override
    public AviatorObject variadicCall(Map<String, Object> env, AviatorObject... args) {
        if (args.length == 0) {
            return AviatorRuntimeJavaType.valueOf(BigDecimal.ZERO);
        }
        BigDecimal sumM = BigDecimal.ZERO;
        //calculate total
        for (AviatorObject arg : args) {
            Object paramM = arg.getValue(env);
            sumM = sumM.add(new BigDecimal(paramM.toString()));
        }

        //calculate average
        BigDecimal average = sumM.divide(BigDecimal.valueOf(args.length), PRECISION, BigDecimal.ROUND_HALF_UP);
        BigDecimal totalB = BigDecimal.ZERO;

        //calculate variance
        for (AviatorObject arg : args) {
            Object param = arg.getValue(env);
            totalB = totalB.add(square(new BigDecimal(param.toString()).subtract(average)));

        }

        //calculate standard deviation
        BigDecimal standardDeviation = totalB.divide(BigDecimal.valueOf(args.length), PRECISION, BigDecimal.ROUND_HALF_UP);

        return AviatorRuntimeJavaType.valueOf(standardDeviation);
    }


    private BigDecimal square(BigDecimal number){
        return number.multiply(number);
    }



    @Override
    public String getName() {
        return "math.variance";
    }
}
```

计算标准差

```java
public class MathStdevFunction extends AbstractVariadicFunction {
    private static final int PRECISION = 14;

    @Override
    public AviatorObject variadicCall(Map<String, Object> env, AviatorObject... args) {
        if (args.length == 0) {
            return AviatorRuntimeJavaType.valueOf(BigDecimal.ZERO);
        }
        BigDecimal sum = BigDecimal.ZERO;
        //calculate total
        for (AviatorObject arg : args) {
            Object param = arg.getValue(env);
            sum = sum.add(new BigDecimal(param.toString()));
        }

        //calculate average
        BigDecimal average = sum.divide(BigDecimal.valueOf(args.length), PRECISION, BigDecimal.ROUND_HALF_UP);
        BigDecimal total = BigDecimal.ZERO;

        //calculate variance
        for (AviatorObject arg : args) {
            Object param = arg.getValue(env);
            total = total.add(square(new BigDecimal(param.toString()).subtract(average)));

        }

        //calculate standard deviation
        BigDecimal standardDeviation = sqrt(total.divide(BigDecimal.valueOf(args.length), PRECISION, BigDecimal.ROUND_HALF_UP));

        return AviatorRuntimeJavaType.valueOf(standardDeviation);
    }


    private BigDecimal square(BigDecimal number){
        return number.multiply(number);
    }

    /**
     * Newton-Raphson method
     * @param value value
     * @return sqrt result
     */
    private static BigDecimal sqrt(BigDecimal value){
        BigDecimal num2 = BigDecimal.valueOf(2);
        int precision = 100;
        MathContext mc = new MathContext(precision, RoundingMode.HALF_UP);
        BigDecimal deviation = value;
        int cnt = 0;
        while (cnt < precision) {
            deviation = (deviation.add(value.divide(deviation, mc))).divide(num2, mc);
            cnt++;
        }
        deviation = deviation.setScale(PRECISION, BigDecimal.ROUND_HALF_UP);
        return deviation;
    }

    @Override
    public String getName() {
        return "math.stdev";
    }
}
```


