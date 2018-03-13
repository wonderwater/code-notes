# snowflake项目阅读

## 1. baidu/uid-generator

[https://github.com/baidu/uid-generator](https://github.com/baidu/uid-generator)

序列规则\(64bits\)：0 \| timestampBits \| workerIdBits \| sequenceBits

位数：1 \| 28 \| 22 \| 13

timestampBits 精确到秒

workerIdBits 由数据库主键提供，每次启动获取

sequenceBits 该秒下的自增序列



项目中用到了缓存，使用ringbuffer缓存下一秒的序列号。

# 



