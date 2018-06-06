[TOC]

### 1.前言

在Java开发中常用的日志工具有Log4j、Log4j2、Apache Commons Log、java.util.logging、slf4j等，这些工具对外的接口不尽相同。为了统一这些接口Mybatis定义了一套统一的日志接口供上层使用，并为上述日志框架提供了相应的日志适配器。

### 2.日志适配器

不同的日志框架提供的Log级别也各不相同。例如java.util.logging提供了9个级别，而Log4j2则有6个级别。Mybatis提供了4个日志级别。

### 3.LogFactory
用来初始化log

### 4.mybatis中的适配器
所有的适配器都实现了Log接口。