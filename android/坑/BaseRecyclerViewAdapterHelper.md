 

使用RecyclerView 的 BaseRecyclerViewAdapterHelper 适配器报如下错误，

```java
Giving up device auth after 5 tries

    java.util.concurrent.ExecutionException: javax.net.ssl.SSLHandshakeException: SSL handshake aborted: ssl=0x7c5b5ad65b08: I/O error during system call, Connection reset by peer

        at kpe.get(SourceFile:86)

```

是因为在设置 item 的时候 type 少添加 或者是写错，

