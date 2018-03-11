# snowflake项目阅读

## 1. baidu/uid-generator

[https://github.com/baidu/uid-generator](https://github.com/baidu/uid-generator)

ringbuffer

# 2. False Sharing问题

# [http://ifeve.com/falsesharing/](http://ifeve.com/falsesharing/)

# [http://ifeve.com/false-shareing-java-7-cn/](http://ifeve.com/false-shareing-java-7-cn/)

按照上述的代码实验

"C:\Program Files\Java\jdk1.8.0\_152\bin\javac.exe" FalseSharing.java & "C:\Program Files\Java\jdk1.8.0\_152\bin\java.exe" FalseSharing

不填充

```
C:\Users\water\Desktop>"C:\Program Files\Java\jdk1.8.0_152\bin\javac.exe" FalseSharing.java & "C:\Program Files\Java\jdk1.8.0_152\bin\java.exe" FalseSharing
duration = 28694381243

C:\Users\water\Desktop>"C:\Program Files\Java\jdk1.8.0_152\bin\javac.exe" FalseSharing.java & "C:\Program Files\Java\jdk1.8.0_152\bin\java.exe" FalseSharing
duration = 29091521777

C:\Users\water\Desktop>"C:\Program Files\Java\jdk1.8.0_152\bin\javac.exe" FalseSharing.java & "C:\Program Files\Java\jdk1.8.0_152\bin\java.exe" FalseSharing
duration = 28481436671
```

填充

```
 C:\Users\water\Desktop>"C:\Program Files\Java\jdk1.8.0_152\bin\javac.exe" FalseSharing.java & "C:\Program Files\Java\jdk1.8.0_152\bin\java.exe" FalseSharing
duration = 22903772935

C:\Users\water\Desktop>"C:\Program Files\Java\jdk1.8.0_152\bin\javac.exe" FalseSharing.java & "C:\Program Files\Java\jdk1.8.0_152\bin\java.exe" FalseSharing
duration = 22629886648

C:\Users\water\Desktop>"C:\Program Files\Java\jdk1.8.0_152\bin\javac.exe" FalseSharing.java & "C:\Program Files\Java\jdk1.8.0_152\bin\java.exe" FalseSharing
duration = 22041379522
```





