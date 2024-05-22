## MySQL索引优化
10~50ms听起来是个很难抵达的标准，但实际大部分走索引查询的语句基本上都能控制在该标准内，那又该如何判断一条SQL会不会走索引呢？这里需要使用一个工具：explain，下面一起来聊一聊。
### 3.1、explain分析工具
在之前的[《索引应用篇》](https://juejin.cn/post/7149074488649318431#heading-14)中曾简单聊到过ExPlain这个工具，它本身是MySQL自带的一个执行分析工具，可使用于select、insert、update、delete、repleace等语句上，需要使用时只需在SQL语句前加上一个explain关键字即可，然后MySQL会对应语句的执行计划列出，比如：
![](https://raw.githubusercontent.com/danmuking/image/main/90eede49a9ec4ecb7bebe5b69b0de0de.webp)
上述这些字段在之前也简单提到过，但并未展开细聊，所以在这里就先对其中的每个字段做个全面详解（MySQL8.0版本中才有12个字段，MySQL5.x版本只有10个字段）。
#### 3.1.1、id字段
这是执行计划的ID值，一条SQL语句可能会出现多步执行计划，所以会出现多个ID值，这个值越大，表示执行的优先级越高，同时还会出现四种情况：

- ID相同：当出现多个ID相同的执行计划时，从上往下挨个执行。
- ID不同时：按照ID值从大到小依次执行。
- ID有相同又有不同：先从大到小依次执行，碰到相同ID时从上往下执行。
- ID为空：ID=null时，会放在最后执行。
#### 3.1.2、select_type字段
当前执行的select语句其具体的查询类型，有如下取值：

- SIMPLE：简单的select查询语句，不包含union、子查询语句。
- PRIMARY：union或子查询语句中，最外层的主select语句。
- SUBQUEPY：包含在主select语句中的第一个子查询，如select ... xx = (select ...)。
- DERIVED：派生表，指包含在from中的子查询语句，如select ... from (select ...)。
- DEPENDENT SUBQUEPY：复杂SQL中的第一个select子查询（依赖于外部查询的结果集）。
- UNCACHEABLE SUBQUERY：不缓存结果集的子查询语句。
- UNION：多条语句通过union组成的查询中，第二个以及更后面的select语句。
- UNION RESULT：union的结果集。
- DEPENDENT UNION：含义同上，但是基于外部查询的结果集来查询的。
- UNCACHEABLE UNION：含义同上，但查询出的结果集不会加入缓存。
- MATERIALIZED：采用物化的方式执行的包含派生表的查询语句。

这个字段主要是说明当前查询语句所属的类型，以及在整条大的查询语句中，当前这个查询语句所属的位置。
#### 3.1.3、table字段
表示当前这个执行计划是基于哪张表执行的，这里会写出表名，但有时候也不一定是物理磁盘中存在的表名，还有可能出现如下格式：

- <derivenN>：基于id=N的查询结果集，进一步检索数据。
- <unionM,N>：会出现在查询类型为UNION RESULT的计划中，表示结果由id=M,N...的查询组成。
- <subqueryN>：基于id=N的子查询结果，进一步进行数据检索。
- <tableName>：基于磁盘中已创建的某张表查询。

一句话总结就是：这个字段会写明，当前的这个执行计划会基于哪个数据集查询，有可能是物理表、有可能是子查询的结果、也有可能是其他查询生成的派生表。
#### 3.1.4、partitions字段
这个字段在早版本的explain工具中不存在，这主要是用来显示分区的，因为后续版本的MySQL中支持表分区，该列的值表示检索数据的分区。
#### 3.1.5、type字段
该字段表示当前语句执行的类型，可能出现的值如下：

- all：全表扫描，基于表中所有的数据，逐行扫描并过滤符合条件的数据。
- index：全索引扫描，和全表扫描类似，但这个是把索引树遍历一次，会比全表扫描要快。
- range：基于索引字段进行范围查询，如between、<、>、in....等操作时出现的情况。
- index_subquery：和上面含义相同，区别：这个是基于非主键、唯一索引字段进行in操作。
- unique_subquery：执行基于主键索引字段，进行in操作的子查询语句会出现的情况。
- index_merge：多条件查询时，组合使用多个索引来检索数据的情况。
- ref_or_null：基于次级(非主键)索引做条件查询时，该索引字段允许为null出现的情况。
- fulltext：基于全文索引字段，进行查询时出现的情况。
- ref：基于非主键或唯一索引字段查找数据时，会出现的情况。
- eq_ref：连表查询时，基于主键、唯一索引字段匹配数据的情况，会出现多次索引查找。
- const：通过索引一趟查找后就能获取到数据，基于唯一、主键索引字段查询数据时的情况。
- system：表中只有一行数据，这是const的一种特例。
- null：表中没有数据，无需经过任何数据检索，直接返回结果。

这个字段的值很重要，它决定了MySQL在执行一条SQL时，访问数据的方式，性能从好到坏依次为：

- 完整的性能排序：null → system → const → eq_ref → ref → fulltext → ref_or_null → index_merge → unique_subquery → index_subquery → range → index → all
- 常见的性能排序：system → const → eq_ref → ref → fulltext → range → index → all

一般在做索引优化时，一般都会要求最好优化到ref级别，至少也要到range级别，也就是最少也要基于次级索引来检索数据，不允许出现index、all这类全扫描的形式。
#### 3.1.6、possible_keys字段
这个字段会显示当前执行计划，在执行过程中可能会用到哪些索引来检索数据，但要注意的一点是：可能会用到并不代表一定会用，在某些情况下，就算有索引可以使用，MySQL也有可能放弃走索引查询。
#### 3.1.7、key字段
前面的possible_keys字段表示可能会用到的索引，而key这个字段则会显示具体使用的索引，一般情况下都会从possible_keys的值中，综合评判出一个性能最好的索引来进行查询，但也有两种情况会出现key=null的这个场景：

- possible_keys有值，key为空：出现这种情况多半是由于表中数据不多，因此MySQL会放弃索引，选择走全表查询，也有可能是因为SQL导致索引失效。
- possible_keys、key都为空：表示当前表中未建立索引、或查询语句中未使用索引字段检索数据。

默认情况下，possible_keys有值时都会从中选取一个索引，但这个选择的工作是由MySQL优化器自己决定的，如果你想让查询语句执行时走固定的索引，则可以通过force index、ignore index的方式强制指定。
#### 3.1.8、key_len字段
这个表示对应的执行计划在执行时，使用到的索引字段长度，一般情况下都为索引字段的长度，但有三种情况例外：

- 如果索引是前缀索引，这里则只会使用创建前缀索引时，声明的前N个字节来检索数据。
- 如果是联合索引，这里只会显示当前SQL会用到的索引字段长度，可能不是全匹配的情况。
- 如果一个索引字段的值允许为空，key_len的长度会为：索引字段长度+1。
#### 3.1.9、ref字段
显示索引查找过程中，查询时会用到的常量或字段：

- const：如果显示这个，则代表目前是在基于主键字段值或数据库已有的常量（如null）查询数据。 
   - select ... where 主键字段 = 主键值;
   - select ... where 索引字段 is null;
- 显示具体的字段名：表示目前会基于该字段查询数据。
- func：如果显示这个，则代表当与索引字段匹配的值是一个函数，如： 
   - select ... where 索引字段 = 函数(值);
#### 3.1.10、rows字段
这一列代表执行时，预计会扫描的行数，这个数字对于InnoDB表来说，其实有时并不够准确，但也具备很大的参考价值，如果这个值很大，在执行查询语句时，其效率必然很低，所以该值越小越好。
#### 3.1.11、filtered字段
这个字段在早版本中也不存在，它是一个百分比值，意味着表中不会扫描的数据百分比，该值越小则表示执行时会扫描的数据量越大，取值范围是0.00~100.00。
#### 3.1.12、extra字段
该字段会包含MySQL执行查询语句时的一些其他信息，这个信息对索引调优而言比较重要，可以带来不小的参考价值，但这个字段会出现的值有很多种，如下：

- Using index：表示目前的查询语句，使用了索引覆盖机制拿到了数据。
- Using where：表示目前的查询语句无法从索引中获取数据，需要进一步做回表去拿表数据。
- Using temporary：表示MySQL在执行查询时，会创建一张临时表来处理数据。
- Using filesort：表示会以磁盘+内存完成排序工作，而完全加载数据到内存来完成排序。
- Select tables optimized away：表示查询过程中，对于索引字段使用了聚合函数。
- Using where;Using index：表示要返回的数据在索引中包含，但并不是索引的前导列，需要做回表获取数据。
- NULL：表示查询的数据未被索引覆盖，但where条件中用到了主键，可以直接读取表数据。
- Using index condition：和Using where类似，要返回的列未完全被索引覆盖，需要回表。
- Using join buffer (Block Nested Loop)：连接查询时驱动表不能有效的通过索引加快访问速度时，会使用join-buffer来加快访问速度，在内存中完成Loop匹配。
- Impossible WHERE：where后的条件永远不可能成立时提示的信息，如where 1!=1。
- Impossible WHERE noticed after reading const tables：基于唯一索引查询不存在的值时出现的提示。
- const row not found：表中不存在数据时会返回的提示。
- distinct：去重查询时，找到某个值的第一个值时，会将查找该值的工作从去重操作中移除。
- Start temporary, End temporary：表示临时表用于DuplicateWeedout半连接策略，也就是用来进行semi-join去重。
- Using MRR：表示执行查询时，使用了MRR机制读取数据。
- Using index for skip scan：表示执行查询语句时，使用了索引跳跃扫描机制读取数据。
- Using index for group-by：表示执行分组或去重工作时，可以基于某个索引处理。
- FirstMatch：表示对子查询语句进行Semi-join优化策略。
- No tables used：查询语句中不存在from子句时提示的信息，如desc table_name;。
- ......

除开上述内容外，具体的可参考[《explain-Extra字段详解》](https://link.juejin.cn?target=http%3A%2F%2Fblog.wingflare.com%2F2019%2F10%2Fy2x1d5wv3q6prl3m.html)，其中介绍了Extra字段可能会出现的所有值，最后基于Extra字段做个性能排序：

- Using index → NULL → Using index condition → Using where → Using where;Using index → Using join buffer → Using filesort → Using MRR → Using index for skip scan → Using temporary → Strart temporary,End temporary → FirstMatch

上面这个排序中，仅列出了一些实际查询执行时的性能排序，对于一些不重要的就没有列出了。
### 3.2、索引优化参考项
在上面咱们简单介绍了explain工具中的每个字段值，字段数量也比较多，但在做索引优化时，值得咱们参考的几个字段为：

- key：如果该值为空，则表示未使用索引查询，此时需要调整SQL或建立索引。
- type：这个字段决定了查询的类型，如果为index、all就需要进行优化。
- rows：这个字段代表着查询时可能会扫描的数据行数，较大时也需要进行优化。
- filtered：这个字段代表着查询时，表中不会扫描的数据行占比，较小时需要进行优化。
- Extra：这个字段代表着查询时的具体情况，在某些情况下需要根据对应信息进行优化。

PS：在explain语句后面紧跟着show warings语句，可以得到优化后的查询语句，从而看出优化器优化了什么。
### 3.3、索引优化实践
上面了解了索引优化时的一些参考项，接着来聊聊索引优化的实践，不过在优化之前要先搞清楚什么是索引优化，其实无非就两点：

- 把SQL的写法进行优化，对于无法应用索引，或导致出现大数据量检索的语句，改为精准匹配的语句。
- 对于合适的字段上建立索引，确保经常作为查询条件的字段，可以命中索引去检索数据。

总归说来说去，也就是要让SQL走索引执行，但要记住：并非走了索引就代表你的执行速度就快，因为如果扫描的索引数据过多，依旧可能会导致SQL执行比较耗时，所以也要参考type、rows、filtered三个字段的值，来看看一条语句执行时会扫描的数据量，判断SQL执行时是否扫描了额外的行记录，综合分析后需要进一步优化到更细粒度的检索。
索引优化其实本质上，也就是遵循前面第二阶段提出的SQL小技巧撰写语句，以及合理的使用与建立索引，对于索引怎么建立和使用才最好，具体可参考[《索引应用篇-建立与使用索引的正确姿势》](https://juejin.cn/post/7149074488649318431#heading-8)。
一般来说，SQL写好了，索引建对了，基本上就已经优化到位了，对于一些无可避免的慢SQL执行，比如复杂SQL的执行、深分页等情况，要么就从业务层面着手解决，要么就接受一定的耗时，毕竟凡事不可能做到十全十美。

## 参考资料
[(十七)SQL优化篇：如何成为一位写优质SQL语句的绝顶高手！ - 掘金](https://juejin.cn/post/7164652941159170078#heading-21)
