# False Sharing问题

# 在多核的机器上，由于一核的cache lines失效导致另一核数据不相干cache lines失效

# [http://ifeve.com/falsesharing/](http://ifeve.com/falsesharing/)

# [http://ifeve.com/false-shareing-java-7-cn/](http://ifeve.com/false-shareing-java-7-cn/)

```java
import java.util.concurrent.atomic.AtomicLong;

public final class FalseSharing
    implements Runnable
{
    public final static int NUM_THREADS = 4; // change
    public final static long ITERATIONS = 500L * 1000L * 1000L;
    private final int arrayIndex;

    private static PaddedAtomicLong[] longs = new PaddedAtomicLong[NUM_THREADS];
    static
    {
        for (int i = 0; i < longs.length; i++)
        {
            longs[i] = new PaddedAtomicLong();
        }
    }

    public FalseSharing(final int arrayIndex)
    {
        this.arrayIndex = arrayIndex;
    }

    public static void main(final String[] args) throws Exception
    {
        final long start = System.nanoTime();
        runTest();
        System.out.println("duration = " + (System.nanoTime() - start));
    }

    private static void runTest() throws InterruptedException
    {
        Thread[] threads = new Thread[NUM_THREADS];

        for (int i = 0; i < threads.length; i++)
        {
            threads[i] = new Thread(new FalseSharing(i));
        }

        for (Thread t : threads)
        {
            t.start();
        }

        for (Thread t : threads)
        {
            t.join();
        }
    }

    public void run()
    {
        long i = ITERATIONS + 1;
        while (0 != --i)
        {
            longs[arrayIndex].set(i);
        }
    }

    // 这段代码的来历可以看4楼的回复
    public static long sumPaddingToPreventOptimisation(final int index)
    {
        PaddedAtomicLong v = longs[index];
        return v.p1 + v.p2 + v.p3 + v.p4 + v.p5 + v.p6;
    }

    public static class PaddedAtomicLong extends AtomicLong
    {
        public volatile long p1, p2, p3, p4, p5, p6 = 7L;
    }
}
```

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

"C:\Program Files\Java\jdk-9\bin\javac.exe" FalseSharing.java & "C:\Program Files\Java\jdk-9\b in\java.exe" FalseSharing

不填充

```
c:\Users\water\Desktop
λ "C:\Program Files\Java\jdk-9\bin\javac.exe" FalseSharing.java & "C:\Program Files\Java\jdk-9\b in\java.exe" FalseSharing
duration = 26199867229

c:\Users\water\Desktop
λ "C:\Program Files\Java\jdk-9\bin\javac.exe" FalseSharing.java & "C:\Program Files\Java\jdk-9\b in\java.exe" FalseSharing
duration = 26198826491

c:\Users\water\Desktop
λ "C:\Program Files\Java\jdk-9\bin\javac.exe" FalseSharing.java & "C:\Program Files\Java\jdk-9\b in\java.exe" FalseSharing
duration = 27540490243
```

填充

```
c:\Users\water\Desktop
λ "C:\Program Files\Java\jdk-9\bin\javac.exe" FalseSharing.java & "C:\Program Files\Java\jdk-9\b in\java.exe" FalseSharing
duration = 23057520162

c:\Users\water\Desktop
λ "C:\Program Files\Java\jdk-9\bin\javac.exe" FalseSharing.java & "C:\Program Files\Java\jdk-9\b in\java.exe" FalseSharing
duration = 21607596278

c:\Users\water\Desktop
λ "C:\Program Files\Java\jdk-9\bin\javac.exe" FalseSharing.java & "C:\Program Files\Java\jdk-9\b in\java.exe" FalseSharing
duration = 22879882025
```



