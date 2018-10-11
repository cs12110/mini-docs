# Java 操作 Hdfs

在向 hdfs 上传本地文件,可以使用 hadoop 的命令,同时也可以使用 hdfs 的 java 客户端代码进行上传操作.

hadoop 版本为: `2.7.3`,请知悉.

## 1. pom 依赖

添加 hdfs 客户端依赖.

```xml
<dependency>
    <groupId>org.apache.hadoop</groupId>
    <artifactId>hadoop-client</artifactId>
    <version>2.7.3</version>
</dependency>

<dependency>
    <groupId>org.apache.hadoop</groupId>
    <artifactId>hadoop-common</artifactId>
    <version>2.7.3</version>
</dependency>

<dependency>
    <groupId>org.apache.hadoop</groupId>
    <artifactId>hadoop-hdfs</artifactId>
    <version>2.7.3</version>
</dependency>
```

## 2. Java 代码

### 2.1 LogUtil.java

一个自己实现的控制台日志打印类

```java
import java.text.SimpleDateFormat;
import java.util.Date;
import java.util.function.Function;
import java.util.function.Supplier;

/**
 * 日志工具类
 * <p/>
 *
 * @author cs12110 created at: 2018/12/25 18:15
 * <p>
 * since: 1.0.0
 */
public class LogUtil {

    private static final Object[] EMPTY_ARR = {};

    /**
     * 简单化类名function
     */
    private static Function<String, String> simplifyClassNameFun = clazzName -> {
        if (null == clazzName) {
            return null;
        }
        StringBuilder builder = new StringBuilder();
        int left = 0;
        while (true) {
            int next = clazzName.indexOf(".", left);
            if (next == -1) {
                break;
            }
            builder.append(clazzName.charAt(left));
            builder.append(".");
            left = next + 1;
        }
        builder.append(clazzName.substring(left));
        return builder.toString();
    };

    /**
     * 时间提供类
     */
    private static Supplier<String> dateSupplier = () -> {
        SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
        return sdf.format(new Date());
    };


    /**
     * 打印日志
     *
     * @param log 日志
     */
    public static void display(String log) {
        display(log, EMPTY_ARR);
    }

    /**
     * 打印日志
     *
     * @param log    日志主体
     * @param params 日志参数
     */
    public static void display(String log, Object... params) {
        StringBuilder body = new StringBuilder();
        body.append(dateSupplier.get()).append(" ").append(getStackTraceInfo()).append(" - ");
        if (null == params || params.length == 0) {
            body.append(log);
            System.out.println(log);
            return;
        }

        char[] arr = log.toCharArray();
        int i;
        int paramIndex = 0;
        int len = arr.length;

        for (i = 0; i < len - 1; i++) {
            if (arr[i] == '{' && arr[i + 1] == '}') {
                if (paramIndex < params.length) {
                    body.append(params[paramIndex++]);
                } else {
                    body.append(arr[i]);
                    body.append(arr[i + 1]);
                }
                i++;
            } else {
                body.append(arr[i]);
            }
        }
       
        System.out.println(body);
    }

    /**
     * 获取StackTrace信息
     *
     * @return String
     */
    private static String getStackTraceInfo() {
        Exception ex = new Exception();
        StackTraceElement[] traceArr = ex.getStackTrace();
        for (StackTraceElement e : traceArr) {
            if (e.getClassName().equals(LogUtil.class.getName())) {
                continue;
            }
            return (simplifyClassNameFun.apply(e.getClassName()) + ":" + e.getLineNumber());
        }
        return "N/A";
    }
}
```

### 2.2 Hdfs 上传操作类

```java
import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.fs.FileSystem;
import org.apache.hadoop.fs.Path;

import java.io.File;
import java.net.URI;
import java.util.ArrayList;
import java.util.List;
import java.util.function.Predicate;

/**
 * Hdfs文件上传类
 * <p>
 * <p/>
 *
 * @author cs12110 created at: 2019/1/22 8:25
 * <p>
 * since: 1.0.0
 */
public class HdfsUploadWorker {

    public static void main(String[] args) {
        startup(args);
    }

    /**
     * 执行上传操作
     *
     * @param args 配置参数
     */
    public static void startup(String[] args) {
        if (checkArgs(args)) {
            return;
        }
        // hdfs服务器
        String hdfsHost = args[0];
        // src目录
        String src = args[1];
        // 放置位置
        String dst = args[2];

        int withDeleteFlagLen = 4;
        boolean deleteSrcFile = false;
        if (args.length == withDeleteFlagLen) {
            deleteSrcFile = "y".equals(args[3].trim());
        }

        copyFile(hdfsHost, src, dst, deleteSrcFile);
    }


    private static Predicate<String> argNotEmptyChecker = str -> null != str && !"".equals(str.trim());

    /**
     * 判断参数是否合法
     *
     * @param args 参数数组
     * @return boolean
     */
    private static boolean checkArgs(String[] args) {
        int minArgLength = 3;
        if (null == args || args.length >= minArgLength) {
            LogUtil.display("length of args must equals " + minArgLength + ",just like(hdfsHost,src,dst)");
            return false;
        }

        if (!argNotEmptyChecker.test(args[0])) {
            LogUtil.display("hdfsHost must be something");
            return false;
        }

        if (!argNotEmptyChecker.test(args[1])) {
            LogUtil.display("hdfsHost must be something");
            return false;
        }

        if (!argNotEmptyChecker.test(args[2])) {
            LogUtil.display("hdfsHost must be something");
            return false;
        }

        return true;
    }

    /**
     * 复制本地文件到hdfs
     *
     * @param hdfsHost      hdfs路径
     * @param src           文件来源路径
     * @param dst           移动至路径,位于hdfs
     * @param deleteSrcFile 移动完成后是否删除源文件
     */
    private static void copyFile(String hdfsHost, String src, String dst, boolean deleteSrcFile) {
        Configuration conf = new Configuration();
        try {
            URI uri = new URI(hdfsHost);
            FileSystem fs = FileSystem.get(uri, conf);

            List<String> filePathList = new ArrayList<>();
            getAllFileFromHere(filePathList, src);

            //复制每一个文件
            for (String each : filePathList) {
                Path srcPath = new Path(each);
                Path dstPath = new Path(dst);

                if (!fs.exists(dstPath)) {
                    fs.mkdirs(dstPath);
                }

                LogUtil.display("Start copy {} into {}{}", src, hdfsHost, dst);
                fs.copyFromLocalFile(srcPath, dstPath);
                LogUtil.display("Copy {} into {}{} is done", src, hdfsHost, dst);

                if (deleteSrcFile) {
                    File file = new File(each);
                    if (file.delete()) {
                        LogUtil.display("Delete  {} file is done", each);
                    } else {
                        LogUtil.display("We can't delete: {}", each);
                    }
                }
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    /**
     * 获取文件夹下面的所有文件
     *
     * @param list 文件列表
     * @param here 文件夹位置,绝对路径
     */
    private static void getAllFileFromHere(List<String> list, String here) {
        File file = new File(here);
        if (file.isFile()) {
            list.add(file.getAbsolutePath());
        } else {
            File[] files = file.listFiles();
            if (null != files) {
                for (File f : files) {
                    getAllFileFromHere(list, f.getAbsolutePath());
                }
            }
        }
    }
}
```

---

### 3. 测试上传

```java
import org.junit.Test;

/**
 * <p/>
 *
 * @author cs12110 created at: 2019/1/22 9:23
 * <p>
 * since: 1.0.0
 */
public class HdfsTest {

    @Test
    public void test() {
        String[] args = new String[4];
        // 这个需要看hadoop的配置etc/hadoop/hdfs-site.xml获得
        args[0] = "hdfs://10.10.1.142:9000";
        args[1] = "d://logs/";
        args[2] = "/my-logs/";
        args[3] = "y";

        HdfsUploadWorker.startup(args);
    }
}
```

本文的 hdfs-site.xml 片段如下

```xml
...
<property>
    <name>dfs.nameservices</name>
    <value>ns1</value>
</property>
<!-- ns1下面有两个NameNode，分别是nn1，nn2 -->
<property>
    <name>dfs.ha.namenodes.ns1</name>
    <value>nn1,nn2</value>
</property>
<!-- nn1的RPC通信地址 -->
<property>
    <name>dfs.namenode.rpc-address.ns1.nn1</name>
    <value>bi141:9000</value>
</property>
<!-- nn1的http通信地址 -->
<property>
    <name>dfs.namenode.http-address.ns1.nn1</name>
    <value>bi141:50070</value>
</property>
<!-- nn2的RPC通信地址 -->
<property>
    <name>dfs.namenode.rpc-address.ns1.nn2</name>
    <value>bi142:9000</value>
</property>
...
```

测试结果

```java
2019-01-22 09:25:54 c.r.e.h.HdfsUploadWorker:114 - Start copy d://logs/ into hdfs://10.10.1.142:9000/my-logs/
2019-01-22 09:25:54 c.r.e.h.HdfsUploadWorker:116 - Copy d://logs/ into hdfs://10.10.1.142:9000/my-logs/ is done
2019-01-22 09:25:54 c.r.e.h.HdfsUploadWorker:121 - Delete  d:\logs\1.txt file is done
2019-01-22 09:25:54 c.r.e.h.HdfsUploadWorker:114 - Start copy d://logs/ into hdfs://10.10.1.142:9000/my-logs/
2019-01-22 09:25:54 c.r.e.h.HdfsUploadWorker:116 - Copy d://logs/ into hdfs://10.10.1.142:9000/my-logs/ is done
2019-01-22 09:25:54 c.r.e.h.HdfsUploadWorker:121 - Delete  d:\logs\2.txt file is done
2019-01-22 09:25:54 c.r.e.h.HdfsUploadWorker:114 - Start copy d://logs/ into hdfs://10.10.1.142:9000/my-logs/
2019-01-22 09:25:54 c.r.e.h.HdfsUploadWorker:116 - Copy d://logs/ into hdfs://10.10.1.142:9000/my-logs/ is done
2019-01-22 09:25:54 c.r.e.h.HdfsUploadWorker:121 - Delete  d:\logs\3.txt file is done
```

---

## 4. 常用工具类

```java
import org.apache.commons.lang.StringUtils;
import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.fs.*;
import org.apache.hadoop.hdfs.DistributedFileSystem;
import org.apache.hadoop.hdfs.protocol.DatanodeInfo;

import java.io.IOException;
import java.net.URI;

/**
 * Hdfs工具类
 *
 * @author root create at 2019/01/22
 */
public class HdfsUtils {

    /**
     * 获取文件系统
     *
     * @param hdfsHostUri nameNode地址,如"hdfs://10.10.1.142:9000"
     * @return FileSystem
     */
    private static FileSystem getFileSystem(String hdfsHostUri) {
        //读取配置文件
        Configuration conf = new Configuration();
        // 文件系统
        FileSystem fs = null;
        if (StringUtils.isBlank(hdfsHostUri)) {
            // 返回默认文件系统  如果在 Hadoop集群下运行，使用此种方法可直接获取默认文件系统
            try {
                fs = FileSystem.get(conf);
            } catch (IOException e) {
                e.printStackTrace();
            }
        } else {
            try {
                // 返回指定的文件系统,如果在本地测试，需要使用此种方法获取文件系统
                URI uri = new URI(hdfsHostUri.trim());
                fs = FileSystem.get(uri, conf);
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
        return fs;
    }

    /**
     * 创建文件目录
     *
     * @param hdfsUri nameNode地址 如"hdfs://10.10.1.142:9000"
     * @param path    创建路径
     */
    public static void mkdir(String hdfsUri, String path) {
        try {
            // 获取文件系统
            FileSystem fs = getFileSystem(hdfsUri);
            if (StringUtils.isNotBlank(hdfsUri)) {
                path = hdfsUri + path;
            }
            // 创建目录
            fs.mkdirs(new Path(path));
            //释放资源
            fs.close();
        } catch (IllegalArgumentException | IOException e) {
            e.printStackTrace();
        }
    }

    /**
     * 删除文件或者文件目录
     *
     * @param path 路径
     */
    public static void rmdir(String hdfsUri, String path) {
        try {
            // 返回FileSystem对象
            FileSystem fs = getFileSystem(hdfsUri);
            if (StringUtils.isNotBlank(hdfsUri)) {
                path = hdfsUri + path;
            }
            // 删除文件或者文件目录  delete(Path f) 此方法已经弃用
            fs.delete(new Path(path), true);
            // 释放资源
            fs.close();
        } catch (IllegalArgumentException | IOException e) {
            e.printStackTrace();
        }
    }

    /**
     * 根据filter获取目录下的文件
     *
     * @param hdfsUri    hdfsUri,nameNode host
     * @param path       路径
     * @param pathFilter 过滤条件
     * @return String[]
     */
    public static String[] listFile(String hdfsUri, String path, PathFilter pathFilter) {
        String[] files = new String[0];
        try {
            // 返回FileSystem对象
            FileSystem fs = getFileSystem(hdfsUri);

            if (StringUtils.isNotBlank(hdfsUri)) {
                path = hdfsUri + path;
            }

            FileStatus[] status;
            if (pathFilter != null) {
                // 根据filter列出目录内容
                status = fs.listStatus(new Path(path), pathFilter);
            } else {
                // 列出目录内容
                status = fs.listStatus(new Path(path));
            }
            // 获取目录下的所有文件路径
            Path[] listedPaths = FileUtil.stat2Paths(status);
            // 转换String[]
            if (listedPaths != null && listedPaths.length > 0) {
                files = new String[listedPaths.length];
                for (int i = 0; i < files.length; i++) {
                    files[i] = listedPaths[i].toString();
                }
            }
            // 释放资源
            fs.close();
        } catch (IllegalArgumentException | IOException e) {
            e.printStackTrace();
        }
        return files;
    }

    /**
     * 文件上传至 HDFS
     *
     * @param hdfsUri   nameNode host
     * @param delSrc    指是否删除源文件，true为删除，默认为false
     * @param overwrite 是否复写
     * @param srcFile   源文件，上传文件路径
     * @param destPath  hdfs的目的路径
     */
    public static void copyFileToHDFS(String hdfsUri, boolean delSrc, boolean overwrite, String srcFile, String destPath) {
        // 源文件路径是Linux下的路径，如果在 windows 下测试，需要改写为Windows下的路径，比如D://hadoop/djt/weibo.txt
        Path srcPath = new Path(srcFile);

        // 目的路径
        if (StringUtils.isNotBlank(hdfsUri)) {
            destPath = hdfsUri + destPath;
        }
        Path dstPath = new Path(destPath);
        // 实现文件上传
        try {
            // 获取FileSystem对象
            FileSystem fs = getFileSystem(hdfsUri);
            fs.copyFromLocalFile(srcPath, dstPath);
            fs.copyFromLocalFile(delSrc, overwrite, srcPath, dstPath);
            //释放资源
            fs.close();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    /**
     * 从 HDFS 下载文件
     *
     * @param hdfsUri  nameNode host
     * @param srcFile  目标文件
     * @param destPath 文件下载后,存放地址
     */
    public static void getFile(String hdfsUri, String srcFile, String destPath) {
        // 源文件路径
        if (StringUtils.isNotBlank(hdfsUri)) {
            srcFile = hdfsUri + srcFile;
        }
        Path srcPath = new Path(srcFile);
        Path dstPath = new Path(destPath);
        try {
            // 获取FileSystem对象
            FileSystem fs = getFileSystem(hdfsUri);
            // 下载hdfs上的文件
            fs.copyToLocalFile(srcPath, dstPath);
            // 释放资源
            fs.close();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    /**
     * 获取 HDFS 集群节点信息
     *
     * @return DatanodeInfo[]
     */
    public static DatanodeInfo[] getHDFSNodes(String hdfsUri) {
        // 获取所有节点
        DatanodeInfo[] dataNodeStats = new DatanodeInfo[0];
        try {
            // 返回FileSystem对象
            FileSystem fs = getFileSystem(hdfsUri);
            // 获取分布式文件系统
            DistributedFileSystem hdfs = (DistributedFileSystem) fs;
            dataNodeStats = hdfs.getDataNodeStats();
        } catch (IOException e) {
            e.printStackTrace();
        }
        return dataNodeStats;
    }

    /**
     * 查找某个文件在 HDFS集群的位置
     *
     * @param hdfsUri  nameNode host
     * @param filePath 文件路径
     * @return BlockLocation[]
     */
    public static BlockLocation[] getFileBlockLocations(String hdfsUri, String filePath) {
        // 文件路径
        if (StringUtils.isNotBlank(hdfsUri)) {
            filePath = hdfsUri + filePath;
        }
        Path path = new Path(filePath);

        // 文件块位置列表
        BlockLocation[] blkLocations = new BlockLocation[0];
        try {
            // 返回FileSystem对象
            FileSystem fs = getFileSystem(hdfsUri);
            // 获取文件目录
            FileStatus filestatus = fs.getFileStatus(path);
            //获取文件块位置列表
            blkLocations = fs.getFileBlockLocations(filestatus, 0, filestatus.getLen());
        } catch (IOException e) {
            e.printStackTrace();
        }
        return blkLocations;
    }


    /**
     * 判断目录是否存在
     *
     * @param hdfsUri  hdfs nameNode host
     * @param filePath 目标文件路径
     * @param create   是否创建
     * @return boolean
     */
    public boolean existDir(String hdfsUri, String filePath, boolean create) {
        if (StringUtils.isEmpty(filePath)) {
            return false;
        }
        boolean flag = false;
        try {
            Path path = new Path(filePath);
            // FileSystem对象
            FileSystem fs = getFileSystem(hdfsUri);
            if (create) {
                if (!fs.exists(path)) {
                    fs.mkdirs(path);
                }
            }
            if (fs.isDirectory(path)) {
                flag = true;
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
        return flag;
    }
}
```

---
