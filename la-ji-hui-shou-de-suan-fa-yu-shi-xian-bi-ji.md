# 垃圾回收的算法与实现笔记

---

## 1. Python的引用计数实现

python的字典对象内存分配：

| function | level |
| :--- |---:|
|PyDict\_new\(\)| 3 |
|PyObject\_GC\_New\(\)|2|
|PyObject\_Malloc\(\)|2|
|new\_arena\(\)|1|
|malloc\(\)|0|


## A. C/C++回顾

数组指针：

```cpp
char (*pa)[4];
```

指针数组：

```cpp
char *arr[4] = {"hello", "world"};
```



