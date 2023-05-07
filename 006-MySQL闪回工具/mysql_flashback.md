- [前言](#前言)
- [闪回原理](#闪回原理)
- [工具推荐](#工具推荐)
  - [MyFlash](#myflash)
  - [mysqlbinlog\_flashback](#mysqlbinlog_flashback)
  - [binlog2sql](#binlog2sql)
- [总结](#总结)
- [参考资料](#参考资料)


### 前言

您应该您应该遇到过因为误操作破坏数据库的问题，比如忘了带WHERE条件的UPDATE、DELETE操作，然后就需要进行传统方式的全量 & 增量恢复。现在，给您介绍一下MySQL中的flashback玩法，也可以做到像Oracle的flashback那样。

目前 MySQL 的 flashback（又称 闪回）一般是利用binlog完成的，能快速完成恢复且无需停机维护。
    

### 闪回原理

本节我们先来介绍一下MySQL binlog flashback的基本工作原理。

MySQL 的 binlog 以 event 的形式，记录了 MySQL 中所有的变更情况，利用 binlog 我们就能够重现所记录的所有操作。MySQL 引入 binlog 主要有两个用途：一是为了主从复制；二是用于备份恢复后需要重新应用部分 binlog，从而达到全备+增备的效果。

> MySQL的binlog共有三种可选格式(binlog_format)
> **statement**: 基于SQL语句的模式,一般来说生成的binlog尺寸较小，但是某些不确定性SQL语句或函数在复制过程可能导致数据不一致甚至出错。
> **row**: 基于数据行的模式，记录的是数据行的完整变化。相对更安全，推荐使用(但通常生成的binlog会比其他两种模式大很多)。
> **mixed**: 混合模式，可以根据情况自动选用statement抑或row模式，是一种过渡模式。
>
⚠️：想要使用 binlog flashback 工具，需要将参数设置为：**binlog_format=ROW**，**binlog_row_image=FULL**

<div align="center"><img src=./picture/what-is-it-006-001.png /></div>

### 工具推荐
目前开源的回滚工具，在使用上一般有两种方式，一种是直接输入 binlogFile 参数，工具直接解析对应的 binlog 文件，生成相应的 SQL 语句；第二种是输入 binlog 所在的 MySQL 实例地址，程序伪装成一个 slave 去拉取对应的 binlog 文件，然后进行解析，最后生成相应的 SQL 语句。**第二种相对来说比较方便，因为在实际的生产环境中，我们只需要在管控机器上放置一个闪回工具即可，无需关注需要回滚的 binlog 的具体位置，同时也便于集成到数据库管控平台上，作为一个生态工具向用户暴露**。

#### MyFlash

[MyFlash](https://github.com/Meituan-Dianping/MyFlash) 是美团开发的一个回滚 DML 操作的工具，该用具是开源的。通过该工具，可以实现 MySQL 数据的闪回。**该工具是第一种，需要输入 binlog 文件**。

``` shell
[root@masterdb binary]# pwd
/root/MyFlash/binary
[root@masterdb binary]# ./flashback --help
 Usage:
   flashback [OPTION?]

Help Options:
   -h, --help                  Show help options

Application Options:
   --databaseNames             databaseName to apply. if multiple, seperate by comma(,)
   --tableNames                tableName to apply. if multiple, seperate by comma(,)
   --start-position            start position
   --stop-position             stop position
   --start-datetime            start time (format %Y-%m-%d %H:%M:%S)
   --stop-datetime             stop time (format %Y-%m-%d %H:%M:%S)
   --sqlTypes                  sql type to filter . support INSERT, UPDATE ,DELETE. if multiple, seperate by comma(,)
   --maxSplitSize              max file size after split, the uint is M
   --binlogFileNames           binlog files to process. if multiple, seperate by comma(,)  
   --outBinlogFileNameBase     output binlog file name base
   --logLevel                  log level, available option is debug,warning,error
   --include-gtids             gtids to process
   --exclude-gtids             gtids to skip
```

从上面提供的参数来看，该闪回工具支持如下特性:
- 支持**提供库表级别**的闪回
- 支持**指定时间和位点**进行闪回
- 支持**指定 SQL 类型**进行闪回
- 支持**支持GTID** 闪回

#### mysqlbinlog_flashback

[mysqlbinlog_flashback](https://github.com/58daojia-dba/mysqlbinlog_flashback) 是 58到家 开源的一个闪回工具，已经在阿里生产环境的 rds 上使用，该工具是第二种使用方式，只需要传入 binlog 文件所在 MySQL 示例的地址即可拉取 binlog 文件。

> 由于该工具已经停止更新维护，这里不再进行详细的介绍。


#### binlog2sql

[binlog2sql](https://github.com/danfengcao/binlog2sql) 是最轻量和小巧的一款闪回工具，该工具是第二种使用方式，只需要传入 binlog 文件所在 MySQL 示例的地址即可拉取 binlog 文件。

### 总结

数据闪回功能已经是数据库生态中的一个基本功能，每一家公司的 DBA 团队都应该熟悉它并且经常对其进行演练，避免突发状况下，恢复数据时操作不当对数据进行了二次破坏，并且有些数据库云厂商已经将闪回公共做到了管控平台上，对用户开放使用，这无疑减轻了 DBA 团队的负担，同时也丰富了数据库生态的功能。

当然每一个数据库的 DBA 团队都有自己熟悉的闪回工具，有些是内部使用，没有开源(比如百度数据库团队内部使用的 **binlog_flash** 工具)，有些是开源的，比如上面列举的这些，无论哪种工具，我们都应该随身携带这样一个闪回工具，并对其能做的事情非常熟悉。

### 参考资料
1. MyFlash: https://github.com/Meituan-Dianping/MyFlash
2. binlog2sql: https://github.com/danfengcao/binlog2sql