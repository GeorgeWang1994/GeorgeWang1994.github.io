
# TLS之指定密码口令

在利用`namp`来检查端口的安全性的时候发现，出现一个弱密码，导致最终的结果为`C`，最终的结果是想要把它变成`A`，因此需要把这个弱密码给干掉；



因为没有指定固定的`ciphers`，就会使用默认的Go提供的默认的密码，那其中出现了弱密码口令，因此查询`tls`的用法，只需要在配置中指定固定的密码即可，改动如下：



```go
config := &tls.Config{
   Certificates: []tls.Certificate{cer},
   // 指定密码口令
   CipherSuites: []uint16{
      tls.TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256,
      tls.TLS_RSA_WITH_AES_128_GCM_SHA256,
      tls.TLS_RSA_WITH_AES_256_CBC_SHA,
      tls.TLS_RSA_WITH_AES_256_GCM_SHA384,
   },
}
```



最终的结果如下：



![image](//wx2.sinaimg.cn/mw690/007FyU7Tly1g2ymxnkuuzj30qo0w8ds6.jpg)



