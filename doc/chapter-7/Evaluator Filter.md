# Evaluator Filter

[Evaluator Filter 官网说明](https://logback.qos.ch/manual/filters.html#evalutatorFilter)

到目前为止 <u>Regular filters</u> 和 <u>Turbo Filters</u> 都已经讲完了。那么再来看下最后一个 评估过滤器。

Evaluator Filter 不同于前面两个过滤器，EvaluatorFilter 是封装一个 EventEvaluator 的通用过滤器。它的过滤规则 `满足一定的条件`。通过定义的条件的真伪来返回 `onMatch ` 和 `onMismatch` 指定的值。

便于后面的说明，先看两个栗子：

栗一：

```xml
<configuration>
  <statusListener class="ch.qos.logback.core.status.OnConsoleStatusListener" />

  <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
    <filter class="ch.qos.logback.core.filter.EvaluatorFilter">
      <evaluator>
        <expression>event.getStatusCode() == 404</expression>
      </evaluator>
      <onMismatch>DENY</onMismatch>
    </filter>
   <encoder><pattern>%h %l %u %t %r %s %b</pattern></encoder>
  </appender>

  <appender-ref ref="STDOUT" />
</configuration>
```

栗二：

```xml
<configuration>
  <statusListener class="ch.qos.logback.core.status.OnConsoleStatusListener" />
  <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
    <filter class="ch.qos.logback.core.filter.EvaluatorFilter">
      <evaluator name="Eval404">
        <expression>
         (event.getStatusCode() == 404)
           &amp;&amp;  <!-- ampersand characters need to be escaped -->
         !(event.getRequestURI().contains(".css"))
        </expression>
      </evaluator>
      <onMismatch>DENY</onMismatch>
    </filter>

   <encoder><pattern>%h %l %u %t %r %s %b</pattern></encoder>
  </appender>

  <appender-ref ref="STDOUT" />
</configuration>
```

这两个栗子什么意思先不说，直接开整！


## GEventEvaluator

[GEventEvaluator 过滤器官网说明](https://logback.qos.ch/manual/filters.html#GEventEvaluator)

GEventEvaluator是一个具体的EventEvaluator实现，它使用任意的Groovy语言布尔表达式作为评估标准。它使用的是 `Groovy语言评估表达式`。

在对配置文件的解释中，评估表达式是动态编译的。我们不需要担心实际的管道系统。我们需要关心的是groovy-language表达式是有效性！

评估表达式作用于当前的日志记录事件。Logback自动将ILoggingEvent类型的当前日志事件插入称为“event”的变量，以及它的缩写为“e”。TRACE, DEBUG, INFO, WARN and ERROR 也被导出到表达式的范围内。因此，`event.level == DEBUG` and `e.level == DEBUG` 是等价的。对于级别上的其他比较运算符，LEVEL 应该用 `toInt()` 操作符转换成整数。

看下面一个栗子：

```xml
<configuration>

  <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
    <filter class="ch.qos.logback.core.filter.EvaluatorFilter">      
      <evaluator class="ch.qos.logback.classic.boolex.GEventEvaluator">
        <expression>
           e.level.toInt() >= WARN.toInt() &amp;&amp;  <!-- Stands for && in XML -->
           !(e.mdc?.get("req.userAgent") =~ /Googlebot|msnbot|Yahoo/ )
        </expression>
      </evaluator>
      <OnMismatch>DENY</OnMismatch>
      <OnMatch>NEUTRAL</OnMatch>
    </filter>
    <encoder>
      <pattern>
        %-4relative [%thread] %-5level %logger - %msg%n
      </pattern>
    </encoder>
  </appender>

  <root level="DEBUG">
    <appender-ref ref="STDOUT" />
  </root>
</configuration>
```

## JaninoEventEvaluator

[JaninoEventEvaluator官网说明](https://logback.qos.ch/manual/filters.html#JaninoEventEvaluator)

JaninoEventEvaluator 与 GEventEvaluator 一样，不同的是它使用的是 java 语言来做评估表达式。与JaninoEventEvaluator相比，GEventEvaluator由于Groovy语言的使用，使用起来更加方便，但是JaninoEventEvaluator通常会（很多）更快地运行等效表达式。

看下下面的评估表达式变量：

|变量	|类型	|说明	|
|:------|------:|:-----:|
|event	|LoggingEvent	|与日志请求相关联的原始日志事件。所有下列变量也可以从事件中获得。例如，`event.getMessage()`返回与下面描述的消息变量相同的字符串值。|
|message	|String	|日志请求的原始消息。当打印日志 `LOGGER.info（"Hello {}"，name）;` 名称被赋值 `"Alice"` , `"Hello {}"` 就是消息。|
|formattedMessage	|String	|在日志请求中格式化的消息。当打印日志 `LOGGER.info（"Hello {}"，name）;` 名称被赋值 `"Alice"` , `"Hello Alice"` 就是消息。|
|logger	|String	|记录器包类名称	|
|loggerContext	|[LoggerContextVO](https://logback.qos.ch/xref/ch/qos/logback/classic/spi/LoggerContextVO.html)	|日志记录事件所属的记录器上下文的受限（值对象）视图。	|
|level	|int	|日志级别的 int 值，默认值 `DEBUG`、`INFO`、`WARN `、`ERROR ` 也是可以获取的，所以 `level > INFO`表达式是正确的。|
|timeStamp	|long	|日志事件时间戳	|
|marker	|Marker	|与日志请求相关联的标记对象。注意，marker	可以是null，所以需要避免NullPointerException。|
|mdc	|Map	|在创建日志事件时，包含所有MDC值的映射。可以使用下列表达式来访问价值：`mdc.get("myKey")`。Map类型是非参数化的，因为Janino不支持泛型。所以，`mdc.get()`返回的类型是客体而不是字符串。要在归还的值上调用字符串方法，它必须被转换为字符串。如 `((String) mdc.get("k")).contains("val")`。|
|throwable	|java.lang.Throwable	|如果没有与事件相关联的异常，那么 `throwable` 变量的值将为null。不过，`throwable`不能被序列化，所以在远程系统上值会一直是NULL。对于未知无关的表达式，应该是用 `throwableProxy`|
|throwableProxy	|[	IThrowableProxy](https://logback.qos.ch/xref/ch/qos/logback/classic/spi/IThrowableProxy.html)	|与日志事件相关联的异常的代理,如果没有与事件相关联的异常，那么 `throwableProxy	` 变量的值将为null。与 `throwable`不同的是，当一个例外与一个事件相关联时，`throwableProxy	` 的值即使在远程系统上也是非空的，甚至在序列化之后也是如此。|

看下面一个栗子：

```xml
<configuration>

  <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
    <filter class="ch.qos.logback.core.filter.EvaluatorFilter">      
      <evaluator> <!-- defaults to type ch.qos.logback.classic.boolex.JaninoEventEvaluator -->
        <expression>return message.contains("billing");</expression>
      </evaluator>
      <OnMismatch>NEUTRAL</OnMismatch>
      <OnMatch>DENY</OnMatch>
    </filter>
    <encoder>
      <pattern>
        %-4relative [%thread] %-5level %logger - %msg%n
      </pattern>
    </encoder>
  </appender>

  <root level="INFO">
    <appender-ref ref="STDOUT" />
  </root>
</configuration>
```

**注意：** `<evaluatorJoran> `在没有 `class`情况下，默认类型类型是 `ch.qos.logback.classic.boolex.JaninoEventEvaluator`。这是Joran隐式地推断出组件类型的少数事件之一。

在没有配置过滤器输出日志如下：

```vim
0    [main] INFO  com.mingrn.logbackFilterEvents - logging statement 0
0    [main] INFO  com.mingrn.logbackFilterEvents - logging statement 1
0    [main] INFO  com.mingrn.logbackFilterEvents - logging statement 2
0    [main] DEBUG com.mingrn.logbackFilterEvents - logging statement 3
0    [main] INFO  com.mingrn.logbackFilterEvents - logging statement 4
0    [main] INFO  com.mingrn.logbackFilterEvents - logging statement 5
0    [main] ERROR com.mingrn.logbackFilterEvents - billing statement 6
0    [main] INFO  com.mingrn.logbackFilterEvents - logging statement 7
0    [main] INFO  com.mingrn.logbackFilterEvents - logging statement 8
0    [main] INFO  com.mingrn.logbackFilterEvents - logging statement 9
```

配置过滤器输出日志如下：

```vim
0    [main] INFO  com.mingrn.logbackFilterEvents - logging statement 0
0    [main] INFO  com.mingrn.logbackFilterEvents - logging statement 1
0    [main] INFO  com.mingrn.logbackFilterEvents - logging statement 2
0    [main] DEBUG com.mingrn.logbackFilterEvents - logging statement 3
0    [main] INFO  com.mingrn.logbackFilterEvents - logging statement 4
0    [main] INFO  com.mingrn.logbackFilterEvents - logging statement 5
0    [main] INFO  com.mingrn.logbackFilterEvents - logging statement 7
0    [main] INFO  com.mingrn.logbackFilterEvents - logging statement 8
0    [main] INFO  com.mingrn.logbackFilterEvents - logging statement 9
```

>仔细看看
>
>`[main] ERROR com.mingrn.logbackFilterEvents - billing statement 6`
>
>这条日志被过滤了！


当然，评估表达式也可以使用 java 块替换：

```java
<evaluator>
  <expression>
    if(logger.startsWith("org.apache.http"))
      return true;

    if(mdc == null || mdc.get("entity") == null)
      return false;

    String payee = (String) mdc.get("entity");

    if(logger.equals("org.apache.http.wire") &amp;&amp; <!-- & encoded as &amp; -->
        payee.contains("someSpecialValue") &amp;&amp;
        !message.contains("someSecret")) {
      return true;
    }

    return false;
  </expression>
</evaluator>
```


## Matchers

[Matchers 官网说明](https://logback.qos.ch/manual/filters.html#matcher)

我们虽然可以通过在 String 类中调用 `matches()` 方法来进行模式匹配，但是每次调用过滤器时都会产生一个全新模式对象的编译成本。

为了消除这种开销，您可以预先定义一个或多个Matcher对象。

一旦定义了matcher，它就可以在评价器表达式中**重复引用**。

看下面这个栗子：

```xml
<configuration debug="true">

  <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
    <filter class="ch.qos.logback.core.filter.EvaluatorFilter">
      <evaluator>        
        <matcher>
          <Name>odd</Name>
          <!-- filter out odd numbered statements -->
          <regex>statement [13579]</regex>
        </matcher>

        <expression>odd.matches(formattedMessage)</expression>
      </evaluator>
      <OnMismatch>NEUTRAL</OnMismatch>
      <OnMatch>DENY</OnMatch>
    </filter>
    <encoder>
      <pattern>%-4relative [%thread] %-5level %logger - %msg%n</pattern>
    </encoder>
  </appender>

  <root level="DEBUG">
    <appender-ref ref="STDOUT" />
  </root>
</configuration>
```

在这个栗子中使用了一个正则表达式 `statement [13579]`，意思是 字面量 `statement ` 后面跟 1、3、5、7、9中的任意一个数字！

不过在过滤器中我们定义的是，如果匹配到就拒绝！所以实际要求匹配的是 **偶数**！

输出日志如下：

```vim
260  [main] INFO  com.mingrn.logbackFilterEvents - logging statement 0
264  [main] INFO  com.mingrn.logbackFilterEvents - logging statement 2
264  [main] INFO  com.mingrn.logbackFilterEvents - logging statement 4
266  [main] ERROR com.mingrn.logbackFilterEvents - billing statement 6
266  [main] INFO  com.mingrn.logbackFilterEvents - logging statement 8
```
