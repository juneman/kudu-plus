# kudu-plus kudu可视化工具
Kudu是为Apache Hadoop平台开发的列式数据库。Kudu拥有Hadoop生态系统应用程序的常见技术属性：它可以商用硬件上运行，可横向扩展，并支持高可用性操作。 

## kudu-plus是什么

kudu-plus是可视化管理kudu的工具，由于kudu虽然是列式数据库，但是可以表达成关系数据库类似的表和字段等信息，某种情况下通过可视化管理更加轻松。kuduplus包括对表和数据的操作约束，可以帮助更好的理解kudu。本工具可用于学习和测试等。

## kudu基础

#### kudu列类型
- 布尔
- 8位有符号整数
- 16位有符号整数
- 32位有符号整数
- 64位有符号整数
- unixtime_micros（Unix时代以来的64位微秒）
- 单精度（32位）IEEE-754浮点数
- 双精度（64位）IEEE-754浮点数
- 十进制（详见十进制类型）
- UTF-8编码字符串（最多64KB未压缩）
- 二进制（最多64KB未压缩）

#### kudu分区
  - 范围分区：
  
    Kudu允许在运行时动态添加和删除范围分区，而不会影响其他分区的可用性。删除分区将删除属于该分区的平板电脑以及其中包含的数据。后续插入到已删除的分区中将失败。可以添加新分区，但它们不得与任何现有范围分区重叠。Kudu允许在单个事务更改表操作中删除和添加任意数量的范围分区。
    动态添加和删除范围分区对于时间序列用例特别有用。随着时间的推移，可以添加范围分区以覆盖即将到来的时间范围。例如，存储事件日志的表可以在每个月开始之前添加月份分区，以便保存即将发生的事件。可以删除旧范围分区，以便根据需要有效地删除历史数据。
    
    - 范围分区的键必须是主键列的一个子集
    - 在没有散列分区的范围分区表中，每个范围分区将恰好对应于一个tablet
    - kudu允许在运行时添加或删除范围分区，而不会影响其他分区的可用性。删除分区将删除属于该分区的tablet以及其中包含的数据。后续插入到已删除的分区的数据将失败。添加的新分区不能与现有的范围分区重叠。
    - 动态添加和删除范围分区对于时间序列用例特别有用。随着时间的推移，可以添加范围分区以覆盖即将到来的时间范围。例如，存储事件日志的表可以在每个月开始之前添加月份分区，以便保存即将发生的事件。可以删除旧范围分区，以便在必要时有效地删除历史数据。
    
  
  - 哈希分区：
  
    散列分区按散列值将行分配到许多存储桶之一。在单级散列分区表中，每个桶只对应一个tablet。在表创建期间设置桶的数量。通常，主键列用作要散列的列，但与范围分区一样，可以使用主键列的任何子集。
    当不需要对表进行有序访问时，散列分区是一种有效的策略。散列分区对于在tablet之间随机传播写入非常有效，这有助于缓解热点和不均匀的tablet大小。
    
    - 哈希分区不允许动态添加和删除
    
  - 优缺点：
  
    散列分区可以最大限度地提高写入吞吐量，而范围分区可以避免无限制的tablet增长问题。这两种策略都可以利用分区修剪来优化不同场景下的扫描。使用多级分区，可以将这两种策略结合起来，以获得两者的好处，同时最大限度地减少每种策略的缺点。
    
  - java操作分区：
  
    查看测试用例部分代码
    
#### kudu主键设计：

- 每个Kudu表必须声明由一列或多列组成的主键。与RDBMS主键一样，Kudu主键强制执行唯一性约束。尝试插入具有与现有行相同的主键值的行将导致重复键错误。
- 主键列必须是非可空的，并且可能不是boolean，float或double类型。
- 在表创建期间设置后，主键中的列集可能不会更改。
- 与RDBMS不同，Kudu不提供自动递增列功能，因此应用程序必须始终在插入期间提供完整的主键。
- 行删除和更新操作还必须指定要更改的行的完整主键。Kudu本身不支持范围删除或更新。
- 插入行后，可能无法更新列的主键值。但是，可以删除行并使用更新的值重新插入。

#### kudu存在的已知限制：
- 列数

  默认情况下，Kudu不允许创建超过300列的表。我们建议使用较少列的架构设计以获得最佳性能。

- 单元格大小

  在编码或压缩之前，单个单元不得大于64KB。在Kudu完成内部复合密钥编码之后，构成复合密钥的单元限制为总共16KB。插入不符合这些限制的行将导致错误返回给客户端。

- 行的大小

  虽然单个单元可能高达64KB，而Kudu最多支持300列，但建议单行不要大于几百KB。

- 有效标识符

  表名和列名等标识符必须是有效的UTF-8序列且不超过256个字节。

- 不可变主键
  
  Kudu不允许您更新一行的主键列。

- 不可更改的主键

  Kudu不允许您在创建表后更改主键列。

- 不可更改的分区

  除了添加或删除范围分区之外，Kudu不允许您在创建后更改表的分区方式。

- 不可改变的列类型
  
  Kudu不允许更改列的类型。

- 分区拆分

  创建表后，无法拆分或合并分区。

- 主键列必须在非主键列之前

- 表的副本为奇数，且不能大于7，在建表时指定，且不可修改
  
## 分支说明

master为主要分支，使用kudu-client1.8.0，但我偶尔发现在某些集群的使用中产生如下错误：

    Caused by: org.apache.kudu.client.NoLeaderFoundException: Master config (192.168.20.133:7051) has no leader.Exceptions received: org.apache.kudu.client.RecoverableException: connection disconnected
		
但是将kudu-client的版本改为1.4.0则不会产生此问题，为了正常使用产生了develop-1.4分支，问题正在研究，给出的打包文件也先基于develop-1.4分支进行打包

## kudu-plus版本功能实现

#### v0.0.1（当前）
- 查看kudu集群所有表
- 创建kudu表
- 删除kudu表
- 重命名kudu表
- 更新kudu表结构：修改非主键列名、修改非主键列默认值、修改非主键列的是否允许为空、新增非主键字段、删除非主键字段
- 查看kudu表分区信息
- 预览kudu表数据
- 编辑kudu表非主键列数据
- 删除kudu表数据行
- 新增kudu表数据行
- 检索kudu表数据添加筛选条件

#### v0.0.2功能（预期）
- 创建kudu表可以添加hash分区和range分区
- 编辑kudu表可以添加和删除range分区
- kudu表导出为MySQL或其他类型导出
- kudu表导入数据

#### 软件截图

![soft1](https://github.com/Xchunguang/kudu-plus/blob/master/src/main/resources/pages/images/soft-1.jpg)

![soft2](https://github.com/Xchunguang/kudu-plus/blob/master/src/main/resources/pages/images/soft-2.jpg)

![soft3](https://github.com/Xchunguang/kudu-plus/blob/master/src/main/resources/pages/images/soft-3.jpg)

#### 下载试用
链接：https://pan.baidu.com/s/1hMzBGyGH6xvsLksolc_FaQ 
提取码：7e7o 

