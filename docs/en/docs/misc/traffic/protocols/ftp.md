# FTP

`FTP` (`File Transfer Protocol`) is one of the protocols in the `TCP/IP` protocol suite. The `FTP` protocol consists of two components: the `FTP` server and the `FTP` client. The `FTP` server is used to store files, and users can use an `FTP` client to access resources on the `FTP` server via the `FTP` protocol. When developing websites, the `FTP` protocol is typically used to upload web pages or programs to the `Web` server. Additionally, because `FTP` has very high transfer efficiency, it is generally used for transferring large files over the network.

By default, the `FTP` protocol uses `TCP` ports `20` and `21`, where `20` is used for data transfer and `21` is used for control information transfer. However, whether port `20` is used for data transfer depends on the transfer mode used by `FTP`. If active mode is used, then the data transfer port is `20`; if passive mode is used, the specific port to be used is negotiated between the server and the client.
