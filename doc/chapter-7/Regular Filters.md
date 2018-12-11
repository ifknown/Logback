

# Regular filters

[Regular filters 官网说明](https://logback.qos.ch/manual/filters.html#filter)

Regular Filter 继承 [Filter](https://logback.qos.ch/xref/ch/qos/logback/core/filter/Filter.html) 抽象类，本质上是由一个单一的 `decide()` 方法组成，该方法只有一个实例参数 `ILoggingEvent `。

Regular filters 被组织成一个有序列表，并且基于三元逻辑。每个过滤器的 `decide(ILoggingEvent event)` 方法依次被调用。这个方法返回一个FilterReply枚举值：

- `DENY` 拒绝
- `NEUTRAL` 中立
- `ACCEPT` 接受

1）如果  `decide()` 返回的值是 `DENY`，那么日志事件就会立即删除，而不需要咨询其余的过滤器。

2）如果返回的值是 `NEUTRAL`，那么将咨询列表中的下一个过滤器。如果没有进一步的过滤器进行协商，那么日志事件通常会被处理。

3）如果返回的值是 `ACCEPT`，那么日志事件将跳过剩余的过滤器。

**Regular Filter** 一般用于 `<appender>` 组件中。通过向appender添加一个或多个过滤器，您可以通过任意的标准来过滤事件，比如日志消息的内容、MDC的内容、每天的时间或日志事件的任何其他部分。


## Implementing your own Regular Filter

[Implementing your own Filter 官网说明](https://logback.qos.ch/manual/filters.html#yourOwnFilter)

要想实现自己的常规过滤器其实不难，之前也说了，Regular 过滤器继承至抽象类 Filter 并实现了 单一方法 `decide()`。

看下面的一个小栗子：

```java
package com.mingrn.filters;

import ch.qos.logback.classic.spi.ILoggingEvent;
import ch.qos.logback.core.filter.Filter;
import ch.qos.logback.core.spi.FilterReply;

public class SampleFilter extends Filter<ILoggingEvent> {

  @Override
  public FilterReply decide(ILoggingEvent event) {    
    if (event.getMessage().contains("sample")) {
      return FilterReply.ACCEPT;
    } else {
      return FilterReply.DENY;
    }
  }
}
```

在这个 java 类中，笔者继承了 Filter 类，重写了 `decide()` 方法。该方法的内容很简单，如果事件消息包含 `sample` 那就接受，否则就拒绝。

那么现在在 `logback-spring.xml` 文件中将该类通过 `<filter>` 标签引入：

```xml
<configuration>
  <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">

    <filter class="com.mingrn.filters.SampleFilter" />

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

很简单的一个配置。通过嘎斯组件在控制台打印日志时 如果日志内容包含 `sample` 文字就打印。否则不打印！


## Level Filter

[Level Filter 官网说明]()

**日志级别过滤器** 根据 Level 匹配过滤事件。如果事件的级别等于配置级别，则过滤器接受或拒绝事件，这取决于  `onMatch` 和 `onMisMatch` 配属性的配置。

看这个栗子：

```xml
<configuration>
  <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
    <filter class="ch.qos.logback.classic.filter.LevelFilter">
      <level>INFO</level>
      <onMatch>ACCEPT</onMatch>
      <onMismatch>DENY</onMismatch>
    </filter>
    <encoder>
      <pattern>
        %-4relative [%thread] %-5level %logger{30} - %msg%n
      </pattern>
    </encoder>
  </appender>
  <root level="DEBUG">
    <appender-ref ref="CONSOLE" />
  </root>
</configuration>
```

在配置文件中 设定 日志组件 输出级别为 `INFO` ，如果日志级别等于该级别就输出。否则就不输出！说到这里，那么在回到我们最初的问题上：如何将不同级别的日志输出到不同级别的日志文件中？现在看到这个栗子是不是知晓了？


## Threshold Filter

[Threshold Filter官网说明](https://logback.qos.ch/manual/filters.html#thresholdFilter)

**阈值过滤器** 类似于 前面讲的 <u>日志级别过滤器 </u>。Threshold Filter 将日志设了一个门槛，当日志级别大于等于设定的日志级别时就保持中立，否则就拒绝！

直接看这个栗子：

```xml
<configuration>
  <appender name="CONSOLE"
    class="ch.qos.logback.core.ConsoleAppender">
    <!-- deny all events with a level below INFO, that is TRACE and DEBUG -->
    <filter class="ch.qos.logback.classic.filter.ThresholdFilter">
      <level>INFO</level>
    </filter>
    <encoder>
      <pattern>
        %-4relative [%thread] %-5level %logger{30} - %msg%n
      </pattern>
    </encoder>
  </appender>
  <root level="DEBUG">
    <appender-ref ref="CONSOLE" />
  </root>
</configuration>
```
