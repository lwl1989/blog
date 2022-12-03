CreateTime:2018-08-23 16:09:46.0

### 什么是JAVA注解

> 注解（Annotation），也叫元数据。一种代码级别的说明。它是JDK1.5及以后版本引入的一个特性，与类、接口、枚举是在同一个层次。它可以声明在包、类、字段、方法、局部变量、方法参数等的前面，用来对这些元素进行说明，注释。

天哪，这真是一个糟糕的决定，反正我认为是，原因：用专业名词来介绍专业名词。

> Java 注解用于为 Java 代码提供元数据。作为元数据，注解不直接影响你的代码执行，但也有一些类型的注解实际上可以用于这一目的。Java 注解是从 Java5 开始添加到 Java 的。

你看到了吗？不直接影响，但是也可能影响。那就是说，沃德天，到底影响还是不影响。

这里有更多解释方便你理解注解，我觉得写得很好 [秒懂，Java 注解 （Annotation）你可以这样学](https://blog.csdn.net/briblue/article/details/73824058/)

最终我也同意博客里对注解的理解，就是标签，一个抽象化的标签。

### 基本名词和应用

#### 注解的定义

```
public @interface Test1 {
}
```
如果你要使用一个注解，只需要调用一下即可
```
@Test1
public class T{
}
```

#### 元注解

计算机里一般 元XX 都代表最基本最原始的东西，因此，元注解就是最基本不可分解的注解。

包含： @Retention、@Documented、@Target、@Inherited、@Repeatable 5 种。

##### @Retention

Retention 的英文意为保留期的意思。当 @Retention 应用到一个注解上的时候，它解释说明了这个注解的的存活时间。

它的取值如下：

- RetentionPolicy.SOURCE 注解只在源码阶段保留，在编译器进行编译时它将被丢弃忽视。
- RetentionPolicy.CLASS 注解只被保留到编译进行的时候，它并不会被加载到 JVM 中。
- RetentionPolicy.RUNTIME 注解可以保留到程序运行的时候，它会被加载进入到 JVM 中，所以在程序运行时可以获取到它们。

所以，我们可以看得出，当它的值为 RUNTIME 时，则可能会影响我们代码的实际运行结果，其他两种情况是不会影响运行结果的。

##### @Documented

顾名思义，这个元注解肯定是和文档有关。它的作用是能够将注解中的元素包含到 Javadoc 中去。

##### @Target

Target 是目标的意思，@Target 指定了注解运用的地方。

它的取值如下：

- ElementType.ANNOTATION_TYPE 可以给一个注解进行注解
- ElementType.CONSTRUCTOR 可以给构造方法进行注解
- ElementType.FIELD 可以给属性进行注解
- ElementType.LOCAL_VARIABLE 可以给局部变量进行注解
- ElementType.METHOD 可以给方法进行注解
- ElementType.PACKAGE 可以给一个包进行注解
- ElementType.PARAMETER 可以给一个方法内的参数进行注解
- ElementType.TYPE 可以给一个类型进行注解，比如类、接口、枚举

##### @Inherited

CSS里也有这个名词，意味着集成上层结构的属性。只不过在JAVA里，是这样子： 假如定义了@Inherited,如果子类没有应用任何注解的话，则它会继承父类的注解。

简而言之就是，这个继承指的是注解的继承，而不是成员和成员函数。

##### @Repeatable（since java8）

Repeatable 是可重复的意思。即表示，注解是可重复的。

在java8之前，也可以实现重复的注解。增加这个注解是为了增加可读性。

old

```
public @interface Person {
   String area();
}
public @interface Persons {
   Person[] value();
}
public class RepeatPersonDemo {
   @Persons({@Person(area="中国人"),@Person(area="美国人")})
   public void do() {

   }
}
```

new
```
@Repeatable(Persons.class)
public @interface Person {
   String area();
}
public @interface Persons {
   Person[] value();
}
public class RepeatPersonDemo {
   @Person(area="中国人")
   @Person(area="美国人")
   public void do() {

   }
}
```

look，just语法糖


备注：

lambda表达式和默认方法 （JEP 126）

批量数据操作（JEP 107）

类型注解（JEP 104）

注：JEP=JDK Enhancement-Proposal (JDK 增强建议 )，每个JEP即一个新特性。


### 注解实战

#### 测试自带注解

```
package Annotations;

public interface SysInterface {

    public String test();

    @Deprecated
    public Integer test1();
}

public class Sys implements SysInterface{

    //Override  重写
    @Override
    public String test() {
        return "你好啊";
    }

    @Override
    public Integer test1() {
        return Integer.valueOf(3);
    }
    //忽略过期警告   接口内指定了 @Deprecated     Deprecation
    @SuppressWarnings("deprecation")
    public static void main(String[] args) {
        Sys sys = new Sys();

        System.out.println(sys.test1());
    }
}
```

#### 自定义注解

首先我们定义自己的注解，参数可以随便调

```
import java.lang.annotation.*;

@Target({ElementType.TYPE,ElementType.FIELD})  //可以在低端和类上使用
@Retention(RetentionPolicy.RUNTIME)  //运行时
@Inherited
@Documented
public @interface Test1 {
    String Annotation1();

    int number() default 12;
}

@Test1(Annotation1 = "你好啊")
public class Self1 {

    @NoMember
    public int T1() {
        return 1;
    }
}

```

测试

```
public static void main(String[] args) {

        System.out.println("Hello World!");

        try {
            Class c = Class.forName("Self.Self1");
            boolean exists = c.isAnnotationPresent(Test1.class);
            if(exists) {
                Test1 test1 = (Test1)c.getAnnotation(Test1.class);
                System.out.println(test1.Annotation1());
            }

            Method[] method = c.getMethods();
            for(Method m : method) {
                boolean mexists = m.isAnnotationPresent(NoMember.class);
                if(mexists) {
                    System.out.println(m.toString()+" is Annotation by " + NoMember.class);
                }
                //System.out.println(mexists+m.toString());
            }
            System.out.println();
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        }
    }
```

运行结果

```
Hello World!
你好啊
public int Self.Self1.T1() is Annotation by interface Self.NoMember
```