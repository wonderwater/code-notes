1. 时间解析问题
```java
new SimpleDateFormat("yyyyMMdd").parse("1994-01-01");
=> 1993-12-31
```
使用apache工具类
```java
DateUtils.parse("1994-01-01", "yyyyMMdd");
=> Exception
```