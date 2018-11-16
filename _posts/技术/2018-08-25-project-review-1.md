---
layout: post
title: 项目总结1- anaDataLoader
category: 技术
tags: DWA Spark ETL 
keywords: DWA Spark ETL
description: 进入数据组后的第一个项目
---

### 背景

应用产生的日志信息，每天都会增量同步到离线环境的`HBase`，即日志表1和日志表2，同时还会将2张表的`rowKey`信息保存在离线环境的`MySQL`中

在该项目中，我负责的内容包括：
1. 将`HBase`日志表1结构化后，存入`MySQL`数据库（T+1）
2. 写一个前端查询页面，提供查询服务（千万数量级的单`MySQL`表）
3. 将查询结果保存成文件推送到FTP服务器指定路径下

#### Part1 HBase2MySQL

**简要逻辑：**

1. 从`MySQL`中读取`rowKey`信息
2. 在`HBase`中scan数据出来
3. 结构化
4. 用`dataFrameToMySQL`将结果写`入MySQL`中
 
**读MySQL**[(squeryl)](http://squeryl.org/inserts-updates-delete.html)
```scala
import org.squeryl.PrimitiveTypeMode._
import org.squeryl.adapters.MySQLAdapter
import org.squeryl.{Session, SessionFactory}

def createSession(): Session = {
    if(session == null) {
        new Session (java.sql.DriverManager.getConnection (connection, username, pw), new MySQLAdapter)
    } else
        session
}

def getRowKey(tableName: String): List[String] = {
    var rowKeyList: List[String] = null
    using(session) {
        val schema = from("tb_rowKey")(rs =>
            where(rs.hbaseTable === tableName)
                select rs.rowKey
        )
        rowKeyList = schema.toList
    }
    rowKeyList
}
```

**scan HBase**
```scala
def getHRDD(sc:SparkContext): NewHadoopRDD[ImmutableBytesWritable, Result] = {
    val hBaseConf = HBaseConfiguration.create()
    hBaseConf.set(TableInputFormat.INPUT_TABLE,"inputTable")
    hBaseConf.set(TableInputFormat.SCAN_ROW_START,"startRowKey")
    hBaseConf.set(TableInputFormat.SCAN_ROW_STOP,"stopRowKey")
    new NewHadoopRDD(sc, classOf[TableInputFormat], classOf[ImmutableBytesWritable], classOf[Result], hBaseConf)
}
```

**dataFrameToMySQL的代码**
```scala
def DFToMySQL(df:DataFrame) = {
    val props = new Properties()
    props.put("user","mysqlUser")
    props.put("password","pw")
    props.put("driver","com.mysql.jdbc.Driver")
    df.write.mode(SaveMode.Append).jdbc("jdbc:mysql://127.0.0.1:3306/myDatabase?.....","tableName",props)
}
// SaveMode: Append, Overwrite, ErrorIfExists, Ignore 针对的是表级别，没有办法实现 insert ignore/ insert if record not exists
```

问题：
1. dataframe的join完select会出现列名重复的情况，需要drop掉或者withColumnRename
2. dataframe中列的顺序需要跟MySQL表一致，[详情](https://www.tuicool.com/articles/qAzYza)

学习了：
1. dataFrame和rdd的基本操作(map/filter/na.fill/)
2. flatMap【flatMap是将一条记录拆成1-N条记录，返回的还是一个list】

#### Part2 select from MySQL

查询服务需要用到的是一张千万数量级的`MySQL`表，和两张维表（每张大概就几百、一千的数据量，用作筛选）。
查询条件是：
1. 查询数量M
2. 筛选条件1 c1
3. 筛选条件2 c2
 
比较麻烦的地方是，业务人员希望每次是抽查的概念，即每次查询都得到不一样的结果集。所以我在这里的思路是：
1. 得到当前筛选条件下大表的总数N
2. 取一个(0,N-M)范围的随机数r
3. select大表时，limit(r,M)
 
```scala
var totalPage:int = 1
var totalCount:int = 20
val queryResult = table.select(r,M,c1,c2)

totalPage = queryResult.size() / 20 + ((queryResult.size() % 20)==0 ? 0 : 1)
totalCount = queryResult.size()

// 得到全部结果后，用myList.subList(startIndex, endIndex)的方法进行分页
var startIndex = (pageIndex - 1) * 20
var endIndex = queryResult.size() - startIndex > 20 ? 20+startIndex : queryResult.size()
```

问题：
1. 查询速度慢(当r很大的时候) 可以考虑用[子查询](http://www.cnblogs.com/zhangzhu/p/3380542.html)的方法，但大表的主键不是顺序主键。
或者用MySQL自己的random()方法。[方法一](http://www.cnblogs.com/phper7/archive/2010/05/26/1744063.html) 
[方法二](https://stackoverflow.com/questions/1823306/mysql-alternatives-to-order-by-rand#)
2. limit从第r条开始可能取不到M条满足条件的结果
3. 因为是将M条记录全部取出放入内存，当M太大时，会占用大量内存


学习了：
1. 自己写了分页逻辑（将每次查询的结果缓存在list中，然后用list.subList(fromIndex,toIndex)的方式返回指定页码的结果）
2. 前端用的是thymeleaf框架，不太习惯（写js的时候，有符号就算转义了也不能用）

#### Part3 Send to FTP

```Java
public static boolean uploadFileToFTP(FTPConf conf, String fileName, String path, String file) {
    FTPClient ftpClient = new FTPClient();
    try{
        ftpClient.connect(conf.getUrl(), conf.getPort());
        ftpClient.login(conf.getUserName(), conf.getPassword());
        ftpClient.setFileType(FTPClient.BINARY_FILE_TYPE);
        
        int reply = ftpClient.getReplyCode();
        if (!FTPReply.isPositiveCompletion(reply)) {
            ftpClient.disconnect();
            LOG.error("connect to FTP fail");
        } else {
            // dir not exist
            if (!ftpClient.changeWorkingDirectory(path)) { 
                LOG.warn(path + " not exist.")
                createFTPDirectory(ftpClient, path);
            }
            InputStream is = new FileInputStream(file);
            ftpClient.storeFile(fileName, is);
            is.close();
            ftpClient.logout();
            
            return true;
        }
    } catch (Exception e) {
        LOG.error("something wrong: ", e);
    }
    
    return false;
    
}
```



#### TODO

1. 优化spark部分代码，提高处理效率
2. 优化查询部分的代码
