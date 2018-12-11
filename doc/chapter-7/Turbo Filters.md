# Turbo Filters

[Turbo Filters官网说明](https://logback.qos.ch/manual/filters.html#TurboFilter)

看过 <u>Regular Filtre</u> 后是不是发现很简单？现在来看下 Turbo Filter。

TurboFilter对象都扩展了 TurboFilter 抽象类。像常规过滤器一样，它们也使用三元逻辑来返回对日志事件的评估。

总的来说，它们的工作方式与前面提到的过滤器非常相似。然而，过滤器和涡轮过滤器之间有两个主要的区别。

- TurboFilter对象与日志上下文绑定在一起。因此，他们不仅仅会被 `appender`使用时 调用，而且每次的日志请求也是会被调用。它们的范围比应用程序附加的过滤器更宽。

-  更重要的是，它们在LoggingEvent对象创建之前被调用。TurboFilter对象不要求日志事件的实例化来过滤日志请求。因此，即使在事件创建之前，涡轮过滤器也适用于对测井事件进行高性能过滤。

## Implementing your own TurboFilter

[Implementing your own TurboFilter 官网说明](https://logback.qos.ch/manual/filters.html#yourOwnTurboFilter)

想实现自己的 Turbo Filter 需要继承 `TurboFilter ` 抽象类，然后重写 `decide()`方法。

看下面一个栗子：

```java
package com.mingrn.logback;

import org.slf4j.Marker;
import org.slf4j.MarkerFactory;

import ch.qos.logback.classic.Level;
import ch.qos.logback.classic.Logger;
import ch.qos.logback.classic.turbo.TurboFilter;
import ch.qos.logback.core.spi.FilterReply;

public class SampleTurboFilter extends TurboFilter {

  String marker;
  Marker markerToAccept;

  @Override
  public FilterReply decide(Marker marker, Logger logger, Level level,
      String format, Object[] params, Throwable t) {

    if (!isStarted()) {
      return FilterReply.NEUTRAL;
    }

    if ((markerToAccept.equals(marker))) {
      return FilterReply.ACCEPT;
    } else {
      return FilterReply.NEUTRAL;
    }
  }

  public String getMarker() {
    return marker;
  }

  public void setMarker(String markerStr) {
    this.marker = markerStr;
  }

  @Override
  public void start() {
    if (marker != null && marker.trim().length() > 0) {
      markerToAccept = MarkerFactory.getMarker(marker);
      super.start();
    }
  }
}
```

自定义的 `SampleTurboFilter` 接受包含特定标记的事件。如果没有找到标记 （`marker`），那么过滤器将责任传递给链中的下一个过滤器。

**注意：**声明的 marker 变量是用于参数传递，自定义涡轮过滤器就是为了用于配置的灵活性，所以要实现 `get()`、`set()`方法。我另外还实现了start（）方法，以检查在配置过程中已经指定了选项。

现在看下怎么在 `logback-spring.xml` 配置文件中使用。

```xml
<configuration>
  <turboFilter class="com.mingrn.logback.SampleTurboFilter">
    <Marker>sample</Marker>
  </turboFilter>

  <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
    <encoder>
      <pattern>
        %-4relative [%thread] %-5level %logger - %msg%n
      </pattern>
    </encoder>
  </appender>

  <root>
    <appender-ref ref="STDOUT" />
  </root>
</configuration>
```

另外 TurboFilter 还有几个过滤器可供使用。

- MDCFilter 检查MDC中给定值的存在，而 DynamicThresholdFilter  允许基于MDC键/级阈值关联的过滤。
- 另一方面，MarkerFilter会检查与日志请求相关联的特定标记的存在。

看下面这个栗子：

```xml
<configuration>

  <turboFilter class="ch.qos.logback.classic.turbo.MDCFilter">
    <MDCKey>username</MDCKey>
    <Value>sebastien</Value>
    <OnMatch>ACCEPT</OnMatch>
  </turboFilter>

  <turboFilter class="ch.qos.logback.classic.turbo.MarkerFilter">
    <Marker>billing</Marker>
    <OnMatch>DENY</OnMatch>
  </turboFilter>

  <appender name="console" class="ch.qos.logback.core.ConsoleAppender">
    <encoder>
      <pattern>%date [%thread] %-5level %logger - %msg%n</pattern>
    </encoder>
  </appender>

  <root level="INFO">
    <appender-ref ref="console" />
  </root>  
</configuration>
```

输出日志如下：

```vim
2016-06-09 15:17:22,859 [main] INFO  com.mingrn.logback - logging statement 0
2016-06-09 15:17:22,875 [main] INFO  com.mingrn.logback - logging statement 1
2016-06-09 15:17:22,875 [main] INFO  com.mingrn.logback - logging statement 2
2016-06-09 15:17:22,875 [main] DEBUG com.mingrn.logback - logging statement 3
2016-06-09 15:17:22,875 [main] INFO  com.mingrn.logback - logging statement 4
2016-06-09 15:17:22,875 [main] INFO  com.mingrn.logback - logging statement 5
2016-06-09 15:17:22,875 [main] INFO  com.mingrn.logback - logging statement 7
2016-06-09 15:17:22,875 [main] INFO  com.mingrn.logback - logging statement 8
2016-06-09 15:17:22,875 [main] INFO  com.mingrn.logback - logging statement 9
```

通过输出的日志可以看出，FilterEvents 应用程序发出10个日志请求，编号为0到9。除了请求3和6之外，所有的请求都是 INFO 级别，与分配给根记录器的请求级别相同。第三个请求，是在调试级别发布的，它低于有效级别。然而，由于MDC的关键 `username` 在第三个请求之前被设置为 `sebastien`，并且在之后删除，MDCFilter 特别接受请求（并且只接受该请求）。第6个请求，在错误级别发布，被标记为 `billing`因此，它被 Marker 拒绝了。

我们可以看到，第三个请求，如果我们只关注总体信息级别，就不应该显示出来，因为它匹配了第一个涡轮过滤器的要求并且被接受了。

另一方面，第6个请求，这是一个错误级别的请求应该已经显示出来了。但它满足了第二个涡轮过滤器，它的OnMatch选项将被拒绝。因此，第6个请求没有显示出来。

## DuplicateMessageFilter

[DuplicateMessageFilter 官网说明](https://logback.qos.ch/manual/filters.html#DuplicateMessageFilter)

`DuplicateMessageFilter(重复过滤器)` 需要单独表示。该过滤器检查重复的消息，如果相同的详细超过一定数量就会删除重复信息。

为了检测重复，该过滤器使用消息之间的简单字符串相等。

看如下栗子：

```
logger.debug("Hello "+name0);
logger.debug("Hello "+name1);
```

假设，如果 `name0` 和 `name1` 是个不相同的两个值，那么 两个消息 `Hello` 就会被认为是不相关的两条信息。

再看这个栗子：

```
logger.debug("Hello {}.", name0);
logger.debug("Hello {}.", name1);
```

通过占位符 `{}`表示的参数化信息，只比较 原始信息，在这个栗子中原始信息是 `Hello {}`，那么他们就是相等的。

>**注意：** 允许重复的次数是由 `AllowedRepetitions ` 属性指定。如，值设为 1 ，那第二条重复日志信息就会被删除。默认值是 5。
为了检测重复，这个过滤器需要在内部缓存中保留对旧消息的引用。这个缓存的大小是由 `CacheSize` 属性决定的。默认情况下，这个设置为100。

看下面一个栗子：

```xml
<configuration>

  <turboFilter class="ch.qos.logback.classic.turbo.DuplicateMessageFilter"/>

  <appender name="console" class="ch.qos.logback.core.ConsoleAppender">
    <encoder>
      <pattern>%date [%thread] %-5level %logger - %msg%n</pattern>
    </encoder>
  </appender>

  <root level="INFO">
    <appender-ref ref="console" />
  </root>  
</configuration>
```

输出日志如下：

```vim
2018-06-09 15:04:26,156 [main] INFO  com.mingrn.logback - logging statement 0
2018-06-09 15:04:26,156 [main] INFO  com.mingrn.logback - logging statement 1
2018-06-09 15:04:26,156 [main] INFO  com.mingrn.logback - logging statement 2
2018-06-09 15:04:26,156 [main] INFO  com.mingrn.logback - logging statement 4
2018-06-09 15:04:26,156 [main] INFO  com.mingrn.logback - logging statement 5
2018-06-09 15:04:26,171 [main] ERROR com.mingrn.logback - billing statement 6
```

`logging statement 0` 是 `logging statement {}` 打印的第一条信息。第二条信息是第一条的重复数据。主要看第四条数据。`logging statement 3` 的日志级别时 DEBUG，有趣的是，它被删除了！

这说明了一个事实：**涡轮过滤器在其他类型的过滤器之前被调用，包括基本的选择规则。**

因此，DuplicateMessageFilter 将 `logging statement 3`认为是重复数据。
