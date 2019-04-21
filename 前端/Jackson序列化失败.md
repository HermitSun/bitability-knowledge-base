### com.fasterxml.jackson.databind.exc.InvalidDefinitionException

今天才研究完注解，就遇到了和注解有关的问题。算是巧合吗？

具体问题是这样的，还是上次那个实例项目，这次又有人打不开了。然后报错是这样的：

```java
com.fasterxml.jackson.databind.exc.InvalidDefinitionException: No serializer found for class com.example.cinema.po.DateLikeForm and no properties discovered to create BeanSerializer (to avoid exception, disable SerializationFeature.FAIL_ON_EMPTY_BEANS) (through reference chain: com.example.cinema.vo.ResponseVO["content"]->java.util.ArrayList[0])
```

这是啥，Jackson序列化失败？前端的报错就一个500，没啥帮助，问题还是出在后端上。你说我一个前端，怎么就过来帮后端debug了呢？算了。既然问题出在Jackson上，那不如先按照编译器的指示，把那个什么`SerializationFeature`关掉试试：

```yaml
spring:
  jackson:
    serialization:
      FAIL_ON_EMPTY_BEANS: false
```

嗯，报错倒是不报错了，前端显示的值却变成了`undefined`。作为前端，我立刻意识到了事情的严重性：后端的数据根本没有传过来，不是什么序列化错误。

检查了一下代码，似乎并没有什么问题，只是我想吐槽一下，这Lombok用得也太懒了，写个Getter/Setter能咋的，不就多两行代码吗？

```java
@Data
public class DateLikeForm {
    // 人数
    private int likeNum;

    // 时间
    private Date likeTime;
}
```

我在网上查的时候，看到有某位老哥说把`private`换成`public`就好了。我有点想笑，这`@Data`的目的不就是为了替代Getter/Setter吗？都有了Getter/Setter，还要什么`public`。等会，Getter/Setter？我突然想起，上一个项目曾经因为没写Getter/Setter带来的不愉快的经历。想着想着，我的脸色就黑了下来。事不宜迟，赶紧试试：

```java
public class DateLikeForm {
    // 人数
    private int likeNum;

    // 时间
    private Date likeTime;

    public int getLikeNum() {
        return likeNum;
    }

    public void setLikeNum(int likeNum) {
        this.likeNum = likeNum;
    }

    public Date getLikeTime() {
        return likeTime;
    }

    public void setLikeTime(Date likeTime) {
        this.likeTime = likeTime;
    }
}
```

OK，问题解决。本质原因就是后端根本没有拿到数据库里的数据。但是问题又来了，为什么用Lombok就拿不到数据呢？我继续研究了下去，然后突然发现启动的时候会有一段文字一闪而过。那是啥？随后，我在message里找到了它：


```java
java: lombok.javac.apt.LombokProcessor could not be initialized. Lombok will not run during this compilation: java.lang.IllegalArgumentException...
```

看样子是Lombok根本就没导入，所以导致了一系列的问题。但是明明能正常启动，IDE也没报错啊？而且，我本地跑得好好的，到了这里咋就出问题了呢？想不通。

再查了一下，发现了[一篇文章](https://www.jianshu.com/p/5b6dc976a4b0)：

”如果 IDEA 中已安装 Lombok 插件，并且在项目中开启了注解处理，就能够正常解析 Lombok 注解。但是很遗憾的是，无法直接在 IDEA 中使用 JDK 9 构建。

解决方法一：降低jdk版本

解决方法二：需采用将构建与运行委托给 Gradle 的方式。“

我的脸再次黑了下来。是这样吗？再一问，果然，用的JDK11。不过为了这个就把Maven换成Gradle，或者降低JDK版本，也太不人道了。一定有什么更好的办法。参考上次遇到的问题，要不换个版本试试？看了一下，项目里用的是这个：

```xml
<dependency>
    <groupId>org.projectlombok</groupId>
    <artifactId>lombok</artifactId>
    <version>1.16.20</version>
</dependency>
```

去Maven仓库查了一下。

![img](file:///C:\Users\syl18\AppData\Roaming\Tencent\Users\595033456\TIM\WinTemp\RichOle\8L@XH6E[B@Q51KNOZO1K{V5.png)

而Java11发布的消息是在2018年3月。2018年1月发布的Lombok能兼容就有鬼了。赶紧换个版本。

OK，问题解决。Java8有什么不好的……