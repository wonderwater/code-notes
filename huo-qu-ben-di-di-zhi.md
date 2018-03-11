# 获取本地地址

InetAddress的使用：

[https://docs.oracle.com/javase/8/docs/api/java/net/InetAddress.html](https://docs.oracle.com/javase/8/docs/api/java/net/InetAddress.html)

```java
// https://github.com/baidu/uid-generator
// com.baidu.fsg.uid.utils.NetUtils
/**
 * Retrieve the first validated local ip address(the Public and LAN ip addresses are validated).
 *
 * @return the local address
 * @throws SocketException the socket exception
 */
public static InetAddress getLocalInetAddress() throws SocketException {
    // enumerates all network interfaces
    Enumeration<NetworkInterface> enu = NetworkInterface.getNetworkInterfaces();

    while (enu.hasMoreElements()) {
        NetworkInterface ni = enu.nextElement();
        if (ni.isLoopback()) {
            continue;
        }

        Enumeration<InetAddress> addressEnumeration = ni.getInetAddresses();
        while (addressEnumeration.hasMoreElements()) {
            InetAddress address = addressEnumeration.nextElement();

            // ignores all invalidated addresses
            if (address.isLinkLocalAddress() || address.isLoopbackAddress() || address.isAnyLocalAddress()) {
                continue;
            }

            return address;
        }
    }

    throw new RuntimeException("No validated local address!");
}
```

## 



