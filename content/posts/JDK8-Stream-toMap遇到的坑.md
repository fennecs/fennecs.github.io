---
title: '[JDK8]Stream toMap遇到的坑'
author: 土川
tags:
  - jdk8
categories:
  - Java基础
  - ''
slug: 2688045267
date: 2018-04-17 11:19:00
draft: true
---
> 略懂略懂。

<!--more-->
# key重复 
写了这么一句代码

	Map<Long, Integer> map = module.getStocks().stream().collect(Collectors.toMap(SkuStockDTO::getSkuId, SkuStockDTO::getSkuStock));
当`SkuStockDTO::getSkuId`拿到相同id时会报`java.lang.IllegalStateException: Duplicate key xxx`  
其实java8已经给我们提供了解决的方式: 方法的第三个参数体现的
![upload successful](/images/pasted-123.png)
第三个参数是一个merge策略
# value为null
value为null的时候是会报空指针，`java.util.stream.Collectors#toMap`其实最终都是调用一个方法
```java
     * @param <T> the type of the input elements
     * @param <K> the output type of the key mapping function
     * @param <U> the output type of the value mapping function
     * @param <M> the type of the resulting {@code Map}
     * @param keyMapper a mapping function to produce keys
     * @param valueMapper a mapping function to produce values
     * @param mergeFunction a merge function, used to resolve collisions between
     *                      values associated with the same key, as supplied
     *                      to {@link Map#merge(Object, Object, BiFunction)}
     * @param mapSupplier a function which returns a new, empty {@code Map} into
     *                    which the results will be inserted
     * @return a {@code Collector} which collects elements into a {@code Map}
     * whose keys are the result of applying a key mapping function to the input
     * elements, and whose values are the result of applying a value mapping
     * function to all input elements equal to the key and combining them
     * using the merge function
     *
     * @see #toMap(Function, Function)
     * @see #toMap(Function, Function, BinaryOperator)
     * @see #toConcurrentMap(Function, Function, BinaryOperator, Supplier)
     */
    public static <T, K, U, M extends Map<K, U>>
    Collector<T, ?, M> toMap(Function<? super T, ? extends K> keyMapper,
                                Function<? super T, ? extends U> valueMapper,
                                BinaryOperator<U> mergeFunction,
                                Supplier<M> mapSupplier) {
        BiConsumer<M, T> accumulator
                = (map, element) -> map.merge(keyMapper.apply(element),
                                              valueMapper.apply(element), mergeFunction);
        return new CollectorImpl<>(mapSupplier, accumulator, mapMerger(mergeFunction), CH_ID);
    }
```
可以看到始终都会调用`Map.merge`，而这个方法有如下代码
```
Objects.requireNonNull(value);
```
解决方法是不用`java.util.stream.Collectors#toMap`，改用下面的姿势
```
Map<Integer, Boolean> collect = list.stream()
        .collect(HashMap::new, (m,v)->m.put(v.getId(), v.getAnswer()), HashMap::putAll);
```
