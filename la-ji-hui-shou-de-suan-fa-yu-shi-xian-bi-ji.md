# 垃圾回收的算法与实现笔记

---

## 1. Python的**引用计数**实现

python的字典对象内存分配：

| function | level |  |
| :--- | ---: | :--- |
| PyDict\_new\(\) | 3 |  |
| PyObject\_GC\_New\(\) | 2 |  |
| PyObject\_Malloc\(\) | 2 |  |
| new\_arena\(\) | 1 | 积蓄0层的空间 |
| malloc\(\) | 0 |  |

0层 - malloc\(\)  
调用时机：大于256byte

1层 -   
arena&gt;pool&gt;block  
以block返回给调用者

unused\_arena\_objects[]: 取可用的area_object
usable\_arenas[]: 取可用的pool。按照area_object.nfreepools降序排序

2层


## A. C/C++回顾

数组指针：

```cpp
char (*pa)[4];
```

指针数组：

```cpp
char *arr[4] = {"hello", "world"};
```

地址对齐hack：根据cpu的不同，但在32位cpu上，glibc malloc总返回8byte对齐的地址，即二进制后三位总为0，这样我们可以利用后三位做标记用途，只是在寻址时将后三位置0


