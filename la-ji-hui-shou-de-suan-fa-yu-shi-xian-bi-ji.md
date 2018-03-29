# 垃圾回收的算法与实现笔记

---

## 1. Python的**引用计数**实现

python的字典对象内存分配：

| function | level ||
| :--- |---:|:--|
|PyDict\_new\(\)| 3 ||
|PyObject\_GC\_New\(\)|2||
|PyObject\_Malloc\(\)|2||
|new\_arena\(\)|1|积蓄0层的空间|
|malloc\(\)|0||

0层 - malloc()
调用时机：大于256byte
1层 - 
arena>pool>block
以block返回给调用者





## A. C/C++回顾

数组指针：

```cpp
char (*pa)[4];
```

指针数组：

```cpp
char *arr[4] = {"hello", "world"};
```



