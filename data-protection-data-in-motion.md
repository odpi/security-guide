# Data Protection: Data In Motion

Encryption is applied to electronic information to ensure its privacy and confidentiality. Wire encryption protects data as it moves into, through, and out of an Hadoop cluster over RPC, HTTP, Data Transfer Protocol \(DTP\), and JDBC:

## Hadoop RPC Encryption

The most common way for a client to interact with a Hadoop cluster is through RPC. A client connects to a NameNode over RPC protocol to read or write a file. RPC connections in Hadoop use the Java Simple Authentication and Security Layer \(SASL\) which supports encryption. When the `hadoop.rpc.protection`property is set to privacy, the data over RPC is encrypted with symmetric keys.

Enable Encrypted RPC by setting the following properties in`core-site.xml`.

```
hadoop.rpc.protection=privacy
```

**Note**: RPC encryption covers not only the channel between a client and a Hadoop cluster but also the inter-cluster communication among Hadoop services.

## Data Transfer Protocol

The NameNode gives the client the address of the first DataNode to read or write the block. The actual data transfer between the client and the DataNode is over Hadoop's Data Transfer Protocol. To encrypt this protocol you must set`dfs.encryt.data.transfer=true`on the NameNode and all DataNodes. The actual algorithm used for encryption can be customized with`dfs.encrypt.data.transfer.algorithm`set to either "3des" or "rc4". If nothing is set, then the default on the system is used \(usually 3DES.\) While 3DES is more cryptographically secure, RC4 is substantially faster.

Enable Encrypted DTP by setting the following properties in`hdfs-site.xml`:

```
dfs.encrypt.data.transfer=true
dfs.encrypt.data.transfer.algorithm=3des
```

`rc4` is also supporter.

Note: Secondary NameNode is not supported with the HTTPS port. It can only be accessed via `http://<SNN>:50090`

## HTTP Encryption

### HTTPS encryption during shuffle

When data moves between the Mappers and the Reducers over the HTTP protocol, this step is called shuffle. Reducer initiates the connection to the Mapper to ask for data; it acts as an SSL client.

See [https://hadoop.apache.org/docs/r2.7.1/hadoop-mapreduce-client/hadoop-mapreduce-client-core/EncryptedShuffle.html](https://hadoop.apache.org/docs/r2.7.1/hadoop-mapreduce-client/hadoop-mapreduce-client-core/EncryptedShuffle.html)

### HTTPS encryption

Users typically interact with Hadoop using a browser or component CLI, while applications use REST APIs or Thrift. Encryption over the HTTP protocol is implemented with the support for SSL across a Hadoop cluster and for the individual components such as Ambari.

You will need to consider encryption for each individual services in your Hadoop cluster. The Hadoop SSL Keystore Factory manages SSL for core services that communicate with other cluster services over HTTP, such as MapReduce, YARN, and HDFS. Other components that have services that are typically not distributed, or only receive HTTP connections directly from clients, use built-in Java JDK SSL tools. Examples include HBase and Oozie.

Consult each service or your distribution document on how to setup HTTPS. You should also consider disabling or blocking HTTP connection if allowed in conjunction with HTTPS.

## JDBC Encryption

HiveServer2 implements encryption with Java SASL protocol’s quality of protection \(QOP\) setting. With this the data moving between a HiveServer2 over JDBC and a JDBC client can be encrypted.

See [https://cwiki.apache.org/confluence/display/Hive/Setting+Up+HiveServer2\#SettingUpHiveServer2-SSLEncryption](https://cwiki.apache.org/confluence/display/Hive/Setting+Up+HiveServer2#SettingUpHiveServer2-SSLEncryption)

