# 第六章 Layouts
[Layouts官网说明](https://logback.qos.ch/manual/layouts.html)

## 什么是布局？

 [什么是布局官网说明](https://logback.qos.ch/manual/layouts.html#WhatIsALayout?)

 说白话就是：**通过布局能将输出的日志转化成一定的格式输出到控制台或者说输出到日志文件中**。

 布局是负责将传入事件转换成字符串的 logback 组件。

 **原理**：布局接口中的 `format()` 方法接受一个代表事件（任何类型）的对象并返回一个字符串。

 看下源码：

```java
public interface Layout<E> extends ContextAware, LifeCycle {

  String doLayout(E event);
  String getFileHeader();
  String getPresentationHeader();
  String getFileFooter();
  String getPresentationFooter();
  String getContentType();
}
```
直面上来看，这个接口很简单，但是已经完全足够满足日常需求了！

## 布局

 [布局官网说明](https://logback.qos.ch/manual/layouts.html#ClassicPatternLayout)

 在实际项目应用中，主要是通过 `<pattern>` 标签进行布局，该标签一般作为 `<encoder>` 的子标签。将布局规则写在 `<pattren>` 标签中，该布局有一定的规则：每个布局转换符必须以 `%` 开始，以可以将这种规则理解为正则。只不过他跟正则毫无关系，说过这种规则你可以在控制台输出任意你想要的日志格式。如果将 进程设为红色，将类用 `[]` 包起来！

 看个简单的栗子：

```xml
<configuration scan="true">
	<property name="FILE_LOG_PATTERN" value="%d{yyyy-MM-dd HH:mm:ss.SSS} ${LOG_LEVEL_PATTERN:-%5p} ${PID:- } --- [%t] %-40.40logger{39} : %m%n${LOG_EXCEPTION_CONVERSION_WORD:-%wEx}" />

	<appender name="ERROR_FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
		<encoder>
			<pattern>${FILE_LOG_PATTERN}</pattern>
		</encoder>

		<rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
			<FileNamePattern>${ERROR_LOG_FILE}.%d{yyyy-MM-dd}.log</FileNamePattern>
			<MaxHistory>60</MaxHistory>
		</rollingPolicy>

		<!-- 只打印ERROR日志 -->
		<filter class="ch.qos.logback.classic.filter.LevelFilter">
			<level>ERROR</level>
			<onMatch>ACCEPT</onMatch>
			<onMismatch>DENY</onMismatch>
		</filter>
	</appender>

	<root level="INFO">
		<appender-ref ref="ERROR_FILE" />
	</root>

</configuration>
```

 在这个栗子中笔者通过 `<property>` 标签定义了一个规则，在 `<appender>` 组件标签中进行引用，当输出错误日志时就会按照笔者定义的规则输出相应的日志格式！

 可能各位读者会奇怪，这个规则到底是怎么定义的？各各字符又代表着什么含义？别急，笔者接下来进行讲解！

 >**注意**：在接下来讲了规则中，在实际使用时一定要在规则符前面加上 `%` ，如 `msg` 字符是输出日志信息的意思，在引用的时候一定要在前面加上 `%`。如 `<pattern>%msg</pattern>`。
