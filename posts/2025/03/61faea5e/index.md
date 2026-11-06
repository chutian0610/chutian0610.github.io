# Java 操作HDFS小结

本文总结了如何使用 Java 客户端操作 HDFS，以及一些注意点。

<!--more-->

## QuickStart

1. 先在项目中引入依赖：

```xml
<dependency>
    <groupId>org.apache.hadoop</groupId>
    <artifactId>hadoop-common</artifactId>
</dependency>
<dependency>
    <groupId>org.apache.hadoop</groupId>
    <artifactId>hadoop-hdfs</artifactId>
</dependency>
```

2. 准备好 HDFS 集群的配置文件

- core-site.xml
- hdfs-site.xml

3. 初始化。

```java
// 设置用户密码

// 方法一
// 启动时通过环境变量设置
// export HADOOP_USER_NAME=xx 
// export HADOOP_USER_PASSWORD=xx

// 方法二
// 代码设置
System.setProperty("HADOOP_USER_NAME", "xx");
System.setProperty("HADOOP_USER_PASSWORD", " xx");

// 加载配置
Configuration conf = new Configuration();
conf.addResource(new Path("/hadoop/hdfs-site.xml"));
conf.addResource(new Path("/hadoop/core-site.xml"));
// 初始化
// FileSystem fs = FileSystem.newInstance(conf);//每次新建一个FileSystem对象
FileSystem fs = FileSystem.get(conf);//只要fs不调用close，一直重用之前创建的FileSystem对象
```

4. 简单使用

```java
/**
 * create a existing file from local filesystem to hdfs
 * @param source
 * @param dest
 * @param conf
 * @throws IOException
 */
public void addFile(String source, String dest, Configuration conf) throws IOException {

  FileSystem fileSystem = FileSystem.get(conf);
  // Get the filename out of the file path
  String filename = source.substring(source.lastIndexOf('/') + 1,source.length());
  // Create the destination path including the filename.
  if (dest.charAt(dest.length() - 1) != '/') {
    dest = dest + "/" + filename;
  } else {
    dest = dest + filename;
  }
  // Check if the file already exists
  Path path = new Path(dest);
  if (fileSystem.exists(path)) {
    System.out.println("File " + dest + " already exists");
    return;
  }
  // Create a new file and write data to it.
  FSDataOutputStream out = fileSystem.create(path);
  InputStream in = new BufferedInputStream(new FileInputStream(new File(
      source)));
  byte[] b = new byte[1024];
  int numBytes = 0;
  while ((numBytes = in.read(b)) > 0) {
    out.write(b, 0, numBytes);
  }
  // Close all the file descriptors
  in.close();
  out.close();
}

/**
 * read a file from hdfs
 * @param file
 * @param conf
 * @throws IOException
 */
public void readFile(String file, Configuration conf) throws IOException {
  FileSystem fileSystem = FileSystem.get(conf);
  Path path = new Path(file);
  if (!fileSystem.exists(path)) {
    System.out.println("File " + file + " does not exists");
    return;
  }
  FSDataInputStream in = fileSystem.open(path);

  String filename = file.substring(file.lastIndexOf('/') + 1,
      file.length());

  OutputStream out = new BufferedOutputStream(new FileOutputStream(
      new File(filename)));

  byte[] b = new byte[1024];
  int numBytes = 0;
  while ((numBytes = in.read(b)) > 0) {
    out.write(b, 0, numBytes);
  }
  in.close();
  out.close();
}

/**
 * delete a directory in hdfs
 * @param file
 * @throws IOException
 */
public void deleteFile(String file, Configuration conf) throws IOException {
  FileSystem fileSystem = FileSystem.get(conf);

  Path path = new Path(file);
  if (!fileSystem.exists(path)) {
    System.out.println("File " + file + " does not exists");
    return;
  }
  fileSystem.delete(new Path(file), true);
}

/**
 * create directory in hdfs
 * @param dir
 * @throws IOException
 */
public void mkdir(String dir, Configuration conf) throws IOException {
  FileSystem fileSystem = FileSystem.get(conf);
  Path path = new Path(dir);
  if (fileSystem.exists(path)) {
    System.out.println("Dir " + dir + " already not exists");
    return;
  }
  fileSystem.mkdirs(path);
}
/**
 * list file status
 *
 * @param path file path
 * @return file status
 */
public List<FileStatus> listFile(String path) throws IOException {
  List<FileStatus> files = new ArrayList<FileStatus>();
  FileSystem fileSystem = FileSystem.get(conf);
  Path fsPath = new Path(path);
  FileStatus[] sts;
  try {
    FileStatus file = fileSystem.getFileStatus(fsPath);
    if (file.isDirectory()) {
      sts = fileSystem.listStatus(fsPath);
    } else {
      sts = new FileStatus[] {file};
    }
  } catch (Exception ex) {
    // may be Wildcard
    sts = getFileSystem().globStatus(fsPath);
  }
  if (sts != null) {
    Collections.addAll(files, sts);
  }
  return files;
}
/**
 * get file size
 *
 * @param fold file path
 * @return file size
 */
public long fileSize(String fold) {
  long len = 0L;
  List<FileStatus> list = listFile(fold.trim());
  for (FileStatus fileStatus : list) {
    len += fileStatus.getLen();
  }
  return len;
}
```

## FileSystem

在上面的初始化阶段可以看到 Filesystem 有两个初始化方式：

- `FileSystem fs = FileSystem.newInstance(conf)`: 每次新建一个FileSystem对象
- `FileSystem fs = FileSystem.get(conf)`: 只要fs不调用close，一直重用之前创建的FileSystem对象(Cache key相同的情况下)

```java
// cache key
static class Key {
final String scheme;
final String authority;
final UserGroupInformation ugi;
// 默认 0，newInstance时会设置为自增 ID
final long unique;
}
```

显然对于方式二，存在多线程共享同一个 FileSystem对象的情况。那么引出了一个问题，FileSystem 是否线程安全？

### HDFS 的并发访问

HDFS由两个主要组件构成：NameNode和DataNode。

- NameNode：负责存储HDFS的元数据，例如文件的目录结构、文件到数据块的映射以及数据块的副本位置。NameNode不直接存储数据块，而是管理数据块的元数据。
- DataNode：负责实际的数据存储和数据块的读写。每个文件被分割成多个数据块，这些数据块分布在集群中的多个DataNode上。

当两个客户端同时尝试访问HDFS中的同一个文件时，具体的行为和机制如下：

- 文件的读取操作
  - 文件读取的机制：HDFS设计为支持高吞吐量的读操作。在文件被读取时，客户端首先向NameNode请求文件的元数据，包括数据块的位置信息。NameNode返回数据块的位置信息后，客户端可以直接从相应的DataNode读取数据块。
  - 并发读取：当两个客户端同时尝试读取同一个文件时，它们会并发地向NameNode请求文件的元数据，并从不同的DataNode上读取数据块。HDFS的设计支持并发读取，因此多个客户端可以同时读取文件而不会发生冲突。数据块的副本机制进一步确保即使某个DataNode出现故障，读取操作也不会受到影响。
- 文件的写入操作
  - 文件写入的机制：当一个客户端向HDFS写入数据时，它会先将数据写入到一组DataNode上，这些DataNode按照HDFS配置的副本数存储数据块的副本。NameNode负责管理文件的元数据和副本的位置。
  - 写入的锁定机制：HDFS在设计上不支持文件的多写并发。也就是说，文件的写入操作是排他的，只有一个客户端可以对文件进行写操作。此时，NameNode会锁定文件，阻止其他客户端进行写入操作。这种锁定机制保证了文件的一致性和完整性，防止数据冲突和损坏。
- 写入和读取的并发
  - 读取和写入的并发：HDFS允许在文件进行写入时，多个客户端仍然可以并发读取文件。这种操作不会影响读取的准确性和一致性。具体来说，当一个客户端对文件进行写入时，其他客户端的读取操作会看到写入之前的文件内容，直到写入完成。这是因为HDFS的写入操作是追加式的，而读取操作基于快照一致性，确保读取到的是文件在某一时刻的一致视图。
  - 写入冲突处理：如果两个客户端尝试同时对同一个文件进行写入，HDFS会处理这种情况。写入操作会被依次执行，第一个客户端的写入操作会先完成，第二个客户端的写入操作将等待第一个操作完成后进行。这种机制避免了并发写入导致的数据冲突问题。
- 文件的修改和删除
  - 文件修改：HDFS对文件的修改操作是通过追加的方式进行的。客户端不能直接修改文件中已有的内容，而是只能在文件末尾追加数据。如果文件被追加数据，所有的读取操作将看到包含追加数据的新版本。
  - 文件删除：当一个客户端请求删除文件时，NameNode会将该文件的元数据标记为删除，并在后台执行文件删除操作。删除操作是全局性的，其他客户端在文件删除后将无法再访问该文件。如果有其他客户端同时访问文件，NameNode会确保删除操作的正确性和一致性，防止删除操作和访问操作的冲突。

### FileSystem 的并发使用

根据 HDFS 的并发访问机制，FileSystem的操作是线程安全的。但是FileSystem实例本身在不同线程间共享，却不是“安全”的。如果有两个线程使用同一个FileSystem实例进行操作，那么在线程任务中一旦调用FileSystem实例的close方法，另一个线程使用这个实例的时候，就会报错。

如果出于某种原因，在不同的线程中使用`FileSystem.get(conf)`获取全新的FileSystem实例，那么需要同时满足两个条件：

- 设置fs.hdfs.impl.disable.cache为true；
- 同时参数要满足：
  - uri的scheme和authority信息必须提供；
  - scheme必须和fs.defaultFS URI的scheme相同；

那么使用时该如何抉择？

- 如果使用创建一个崭新的FileSystem实例，那么必须要记住 close。但是会带来性能上的问题。
- 如果使用 FileSystem 中缓存的实例，可以不 close，FileSystem 中的缓存会通过hadoop 的ShutdownHookManager在服务结束时统一回收。

如果使用了缓存功能，且有非常的多的 Cache Key 时，可能就需要每次创建一个崭新的FileSystem实例。否则应当使用FileSystem中的缓存。

#### FileSystem Pool

如果不使用缓存，也可以通过连接池方式来管理崭新的FileSystem实例，降低性能指标上的毛刺。

```java
public class FileSystemFactory implements PooledObjectFactory<FileSystem> {

    private final Configuration conf;

    public FileSystemFactory(Configuration conf){
        this.conf = conf;
    }

    @Override
    public PooledObject<FileSystem> makeObject() throws Exception {
        return new DefaultPooledObject<FileSystem>(FileSystem.get(conf));
    }

    @Override
    public void destroyObject(PooledObject<FileSystem> pooledObject) throws Exception {
        FileSystem hdfs = pooledObject.getObject();
        hdfs.close();
    }

    @Override
    public boolean validateObject(PooledObject<FileSystem> pooledObject) {
        FileSystem hdfs = pooledObject.getObject();
        try {
            return  hdfs.exists(new Path("/"));
        } catch (IOException e) {
            return false;
        }
    }
    @Override
    public void activateObject(PooledObject<FileSystem> pooledObject) throws Exception {}

    @Override
    public void passivateObject(PooledObject<FileSystem> pooledObject) throws Exception {}}
}
public class HdfsPoolConfig extends GenericObjectPoolConfig<FileSystem> {
  public HdfsPoolConfig HdfsPoolConfig(
    Integer maxTotal,Integer maxIdle,Integer minIdle,
    Long maxWaitMillis,Long minEvictableIdleTimeMillis,
    Long timeBetweenEvictionRunsMillis
  ){
    HdfsPoolConfig hdfsPoolConfig = new HdfsPoolConfig();
    hdfsPoolConfig.setMaxTotal(maxTotal);
    hdfsPoolConfig.setMaxIdle(maxIdle);
    hdfsPoolConfig.setMinIdle(minIdle));
    hdfsPoolConfig.setMaxWaitMillis(maxWaitMillis);
    hdfsPoolConfig.setMinEvictableIdleTimeMillis(minEvictableIdleTimeMillis);
    hdfsPoolConfig.setTimeBetweenEvictionRunsMillis(timeBetweenEvictionRunsMillis);
    return hdfsPoolConfig;
  }
}
public class HdfsPool extends GenericObjectPool<FileSystem> {
    public HdfsPool(PooledObjectFactory<FileSystem> factory) {
        super(factory);
    }
    public HdfsPool(PooledObjectFactory<FileSystem> factory, GenericObjectPoolConfig<FileSystem> config) {
        super(factory, config);
    }
    public HdfsPool(PooledObjectFactory<FileSystem> factory, GenericObjectPoolConfig<FileSystem> config, AbandonedConfig abandonedConfig) {
        super(factory, config, abandonedConfig);
    }
}
```

使用方式如下：

```java
public boolean deleteFile(String path) throws WorkException {
  FileSystem hdfs = null;
  try {
      hdfs = hdfsPool.borrowObject();
      return hdfs.delete(new Path(path), false);
  } catch (Exception e) {
      log.error("删除hdfs文件异常：", e);
      return false;
  } finally {
      if (null != hdfs) {
          hdfsPool.returnObject(hdfs);
      }
  }
}
```

