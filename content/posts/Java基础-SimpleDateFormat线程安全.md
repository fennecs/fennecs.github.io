---
tags:
  - multithread
categories:
  - Java基础
title: '[Java基础]SimpleDateFormat线程安全的问题'
slug: 3683368056
date: 2018-12-25 18:19:00
draft: true
---
> 今天把`SimpleDateFormat`设置为`static`，老哥说你错了，你真的错了。

<!--more-->
# 前言
`SimpleDateFormat`是个线程不安全的类，不可以在多线程里面使用。

# 代码
```java
    private static int num = 4;
    private static ExecutorService executorService = Executors.newFixedThreadPool(num);
    private static SimpleDateFormat simpleDateFormat = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
    public static void main(String[] args) {
        IntStream.range(0, num).parallel().forEach(a -> executorService.submit(() -> {
            try {
                System.out.println(simpleDateFormat.parse("2017-12-13 15:17:27"));
            } catch (Exception e) {
                e.printStackTrace();
            }
        }));

        executorService.shutdown();

    }
```
这个程序会这么报：
```java
java.lang.NumberFormatException: multiple points
	at java.base/jdk.internal.math.FloatingDecimal.readJavaFormatString(FloatingDecimal.java:1890)
	at java.base/jdk.internal.math.FloatingDecimal.parseDouble(FloatingDecimal.java:110)
	at java.base/java.lang.Double.parseDouble(Double.java:543)
	at java.base/java.text.DigitList.getDouble(DigitList.java:169)
	at java.base/java.text.DecimalFormat.parse(DecimalFormat.java:2128)
	at java.base/java.text.SimpleDateFormat.subParse(SimpleDateFormat.java:2240)
	at java.base/java.text.SimpleDateFormat.parse(SimpleDateFormat.java:1541)
	at java.base/java.text.DateFormat.parse(DateFormat.java:393)
	at com.htc.learning.main.SimpleDateFormatTest.lambda$main$0(SimpleDateFormatTest.java:18)
	at java.base/java.util.concurrent.Executors$RunnableAdapter.call(Executors.java:515)
	at java.base/java.util.concurrent.FutureTask.run(FutureTask.java:264)
	at java.base/java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1128)
	at java.base/java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:628)
	at java.base/java.lang.Thread.run(Thread.java:834)
java.lang.NumberFormatException: multiple points
	at java.base/jdk.internal.math.FloatingDecimal.readJavaFormatString(FloatingDecimal.java:1890)
	at java.base/jdk.internal.math.FloatingDecimal.parseDouble(FloatingDecimal.java:110)
	at java.base/java.lang.Double.parseDouble(Double.java:543)
	at java.base/java.text.DigitList.getDouble(DigitList.java:169)
	at java.base/java.text.DecimalFormat.parse(DecimalFormat.java:2128)
	at java.base/java.text.SimpleDateFormat.subParse(SimpleDateFormat.java:2240)
	at java.base/java.text.SimpleDateFormat.parse(SimpleDateFormat.java:1541)
	at java.base/java.text.DateFormat.parse(DateFormat.java:393)
	at com.htc.learning.main.SimpleDateFormatTest.lambda$main$0(SimpleDateFormatTest.java:18)
	at java.base/java.util.concurrent.Executors$RunnableAdapter.call(Executors.java:515)
	at java.base/java.util.concurrent.FutureTask.run(FutureTask.java:264)
	at java.base/java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1128)
	at java.base/java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:628)
	at java.base/java.lang.Thread.run(Thread.java:834)
Wed Dec 13 15:17:27 CST 2017
Tue Dec 13 15:17:27 CST 12
```
这段代码有时报错，有时能正常输出，但也不是正确输出。

# SimpleDateFormat的类图结构

![upload successful](/images/pasted-159.png)
可以看到内部维护了一个Calendar类，归根到底就是这个家伙线程不安全。

# 源码

看parse(String)方法，
```java
    public Date parse(String text, ParsePosition pos){
        // 解析字符串将每步的执行结果放入CalendarBuilder的实例calb中
        ...
        Date parsedDate;
        try {
            parsedDate = calb.establish(calendar).getTime();
            // If the year value is ambiguous,
            // then the two-digit year == the default start year
            if (ambiguousYear[0]) {
                if (parsedDate.before(defaultCenturyStart)) {
                    parsedDate = calb.addYear(100).establish(calendar).getTime();
                }
            }
        }
        // An IllegalArgumentException will be thrown by Calendar.getTime()
        // if any fields are out of range, e.g., MONTH == 17.
        catch (IllegalArgumentException e) {
            ...
        }

        return parsedDate;
    }
```
看`calb.establish(calendar).getTime()`,这里传入的是一个成员变量，每个`SimpleDateFormat`使用一个`Calendar`实例

```java
    Calendar establish(Calendar cal) {
        ...
        // reset日期对象cal的属性值
        cal.clear();
        // 使用calb中中属性设置cal
        ...
        // 
        return cal;
```
由于多个线程使用的是同一个`Calendar`，就会出现一些奇奇怪怪的错误。

那么`format()`呢
```java
public StringBuffer format(Date date, StringBuffer toAppendTo,
                               FieldPosition pos)
    {
        pos.beginIndex = pos.endIndex = 0;
        return format(date, toAppendTo, pos.getFieldDelegate());
    }

    // Called from Format after creating a FieldDelegate
    private StringBuffer format(Date date, StringBuffer toAppendTo,
                                FieldDelegate delegate) {
        // Convert input date to time field list
        calendar.setTime(date);

        ...
        return toAppendTo;
    }
```
主要看这里`calendar.setTime(date);`,同样的原因，calendar被并发操作，最后多个线程会输出同样的值。

# 解决方案
1. 每个线程一个`SimpleDateFormat`实例，不过我觉得这样没必要。
1. 使用Apache的`FastDateFormat`类，这是一个线程安全类

`FastDateFormat`也是依赖`Calendar`，不过每次方法调用都会实例化一次，避免多线程操作同一个`Calendar`。