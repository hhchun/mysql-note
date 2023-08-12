# 存储结构

* 磁盘与内存交互的基础单位：页。

* InnoDB将数据划分为若干个页，页的默认的大小为16KB。

* 因为页是数据库管理存储空间的基本单位，也是数据库I/O操作的最小单位，所以在读取数据时，不管读取的是一行还是多行，都是将行所在的页进行加载；如果是写操作，也是会对数据所在页进行操作，向磁盘进行同步（刷盘操作）时，也是会对整个页进行同步。

* 不同的数据页可能存储在磁盘中不同的位置，可能是**不连续**的，页与页之间是以**双向链表**的方式进行关联的。每个数据页中，不同的记录会按照主键从小到大的顺序组成一个**单向链表**。每个数据页都会为页中存储的记录生成一个**页目录**，方便使用**二分法**快速定位到数据对应的槽，然后再遍历该槽对应分组中的记录即可快速找到指定的记录。

* 页大小的查看

  ```mysql
  show variables like '%innodb_page_size%';
  ```

  > 不同的数据库管理系统（简称DBMS）的页大小不同。比如在MySQL的InnoDB存储引擎中，默认页的大小是**16KB**。
  >
  > 页大小是可以进行修改的，修改变量或者修改配置文件即可；页大小建议修改为2^n。

* 在数据库中，页的上层结构还有区（Extent）、段（Segment）和表空间（TableSpace）的概念。

  <img src="https://hhc-typora.oss-cn-shenzhen.aliyuncs.com/%E5%AD%98%E5%82%A8%E7%BB%93%E6%9E%84.svg" alt="存储结构" style="zoom:125%;" />

  > 1Extent = 64 * 16KB = 1MB
  >
  > 段（Segment）由一个或多个区组成，区在文件系统是一个连续分配的空间（在InnoDB中是连续的64个页)，不过在段中不要求区与区之间是相邻的。**段是数据库中的分配单位，不同类型的数据库对象以不同的段形式存在**。当创建数据表、索引的时候，就会相应创建对应的段。
  >
  > 表空间（Tablespace)是一个逻辑容器，表空间存储的对象是段，在一个表空间中可以有一个或多个段，但是一个段只能属于一个表空间。数据库由一个或多个表空间组成，表空间从管理上可以划分为系统表空间，**用户表空间、撤销表空间、临时表空间**等。

# 页的内部结构

* 页的类型分类：数据页（保存B+树节点）、系统页、Undo页和事务数据页等；数据页是最常使用的页。

* 数据页的存储空间划分为七部分（共16KB）：文件头（File Header）、页头（Page Header）、最大最小记录（Infimum + supremum)、用户记录（User Records）、空闲空间（Free Space）、页目录（Page Directory）和文件尾（File Tailer）。

  | 名称             | 占用大小 | 说明                                               |
  | ---------------- | -------- | -------------------------------------------------- |
  | File Header      | 38字节   | 文件头，描述页的信息（页的编号、其上一页、下一页） |
  | Page Header      | 56字节   | 页头，页的状态信息                                 |
  | lnfimum-Supremum | 26字节   | 最大和最小记录，这是两个虚拟的行记录               |
  | User Records     | 不确定   | 用户记录，存储行记录内容                           |
  | Free Space       | 不确定   | 空闲记录，页中还没有被使用的空间                   |
  | Page Directory   | 不确定   | 页目录，存储用户记录的相对位置                     |
  | File Trailer     | 8字节    | 文件尾，校验页是否完整                             |

* 第一部分：File Header（文件头部）、File Trailer（文件尾部）。

  * File Header：

    * FIL_PAGE_OFFSET（4字节）：页号；每个页都有一个唯一的页号。

    * FIL_PAGE_TYPE（2字节）：页的类型。

      | 类型名称                | 十六进制 | 描述                                |
      | ----------------------- | -------- | ----------------------------------- |
      | FIL_PAGE_TYPE_ALLOCATED | 0x0000   | 最新分配，还没使用                  |
      | **FIL_PAGE_UNDO_LOG**   | 0x0002   | Undo日志页                          |
      | FIL_PAGE_INODE          | 0x0003   | 段信息节点                          |
      | FIL_PAGE_IBUF_FREE_LIST | 0x0004   | Insert Buffer空闲列表               |
      | FIL_PAGE_IBUF_BITMAP    | 0x0005   | Insert Buffer位图                   |
      | **FIL_PAGE_TYPE_SYS**   | 0x0006   | 系统页                              |
      | FIL_PAGE_TYPE_TRX_SYS   | 0x0007   | 事务系统数据                        |
      | FIL_PAGE_TYPE_FSP_HDR   | 0x0008   | 表空间头部信息                      |
      | FIL PAGE_TYPE_XDES      | 0x0009   | 扩展描述页                          |
      | FIL_PAGE_TYPE_BLOB      | 0x000A   | 溢出页                              |
      | **FIL_PAGE_INDEX**      | 0x45BF   | 索引l页，也就是我们所说的**数据页** |

    * FIL_PAGE_PREV（4字节）和 FIL_PAGE_NEXT（4字节）：上一页的页号和下一页的页号。

      <img src="https://hhc-typora.oss-cn-shenzhen.aliyuncs.com/FIL_PAGE_PREV.svg" alt="FIL_PAGE_PREV" style="zoom:150%;" />

    * FIL_PAGE_SPACE_OR_CHKSUM（4字节）：当前页校验和；文件头部和文件尾部都有这个属性；用于保证页的完整性。

    * FIL_PAGE_LSN（8字节）：页最后修改时对应的日志序列位置（LSN）。

  * File Trailer：前4个字节代表页的校验和；页最后修改时对应的日志序列位置（LSN）；用于校验页的完整性，如果文件头部和文件尾部的LSN值校验不成功的话，就说明同步过程出现了问题。

* 第二部分：Free Space（空闲空间）、User Records（用户记录）、Infimum + Supremum（最小最大记录）

  * Free Space：当存储记录时会按照指定的**行格式**存储到User Records中，在生成页的时候其实并没有User Records，而是**当每插入一条记录时从Free Space中申请此记录大小的空间到User Records中**。当Free Space的空间被User Records申请完之后，意味着页已被使用完毕，如果还有新记录插入，则需要去**申请新的页**。

    <img src="https://hhc-typora.oss-cn-shenzhen.aliyuncs.com/Free%20Space.svg" alt="Free Space" style="zoom:150%;" />

  * User Records：用于存储插入的记录；按照指定的行格式进行存储，记录与记录之间使用单链表进行关联。详细的内容请参考**行格式的记录头信息**。

  * Infimum + Supremum：最小最大记录；记录的大小是根据主键进行比较的。这两条记录并不存放在User Records中，而是单独存储在 Infimum + Supremum 中，由MySQL自动插入（称为伪记录或虚拟记录）。

    <img src="https://hhc-typora.oss-cn-shenzhen.aliyuncs.com/Infimum%20Supremum.svg" alt="Infimum Supremum"  />

* 第三部分：Page Directory（页目录）、Page Header（页面头部）

  * Page Directory

    * 为什么需要页目录？

      答：在页中，记录是单向链表的形式进行存储的，检索效率不高，因此专门做一个目录，方便二分查找。

    * 页目录的设计：

      1. 将页中所有的记录（包括最小最大记录，不包括已删除记录）进行分组。

      2. 第一组只有最小记录；最后一组包含最大记录，数量为1-8条；其余组记录数量为4-8条之间；除第一组外，其余组的记录数量**尽量平分**。

      3. 每组中最后一条记录的**头信息**中都会存储当前组共有多少条记录，字段为**n_owned**。

      4. 页目录用来存储每组最后一条记录的**地址偏移量**，这些地址偏移量会按照先后顺序存储起来，每组的地址偏移量也被称之为**槽（slot）**，每个槽相当于指针指向了不同组的最后一个记录。

         ![Page Directory2](https://hhc-typora.oss-cn-shenzhen.aliyuncs.com/Page%20Directory2.svg)

  * Page Header：存储页中各种状态信息。

    | 名称              | 占用空间大小（字节） | 描述                                                         |
    | ----------------- | -------------------- | ------------------------------------------------------------ |
    | PAGE_N_DIR_SLOTS  | 2                    | 在页目录中的槽数量                                           |
    | PAGE_HEAP_TOP     | 2                    | 还未使用的空间最小地址，也就是说从该地址之后就是**Free space** |
    | PAGE_N_HEAP       | 2                    | 本页中的记录的数量（包括最小和最大记录以及标记为删除的记录） |
    | PAGE_FREE         | 2                    | 第一个已经标记为删除的记录地址（各个已删除的记录通过 next_record也会组成一个单链表，这个单链表中的记录可以被重新利用） |
    | PAGE_GARBAGE      | 2                    | 已删除记录占用的字节数                                       |
    | PAGE_LAST_INSERT  | 2                    | 最后插入记录的位置                                           |
    | PAGE_DIRECTION    | 2                    | 记录插入的方向                                               |
    | PAGE_N_DIRECTION  | 2                    | 一个方向连续插入的记录数量                                   |
    | PAGE_N_RECS       | 2                    | 该页中记录的数量（不包括最小和最大记录以及被标记为删除的记录） |
    | PAGE_MAX_TRX_ID   | 8                    | 修改当前页的最大事务ID，该值仅在二级素引中定义               |
    | PAGE_LEVEL        | 2                    | 当前页在B+树中所处的层级                                     |
    | PAGE_INDEX_ID     | 8                    | 索引ID，表示当前页属于哪个素引                               |
    | PAGE_BTR_SEG_LEAF | 10                   | B+树叶子段的头部信息，仅在B+树的Root页定义                   |
    | PAGE_BTR_SEG_TOP  | 10                   | B+树非叶子段的头部信息，仅在B+树的Root页定义                 |
    
    * PAGE_DIRECTION：假如新插入的一条记录的主键值比上一条记录的主键值大，我们说这条记录的插入方向是右边，反之则是左边。用来表示最后一条记录插入方向的状态就是PAGE_DIRECTION。
    * PAGE_N_DIRECTION：假设连续几次插入新记录的方向都是一致的，InnoDB会把沿着同一个方向插入记录的条数记下来，这个条数就用PAGE_N_DIRECTION这个状态表示。当然，如果最后一条记录的插入方向改变了的话，这个状态的值会被清零重新统计。

# 行格式

## 概述

* InnoDB行格式类型：Compact（紧密）、Redundant（冗余）、Dynamic（动态）和Compressed（压缩）。

  ```mysql
  # 查询默认行格式
  select @@innodb_default_row_format;
  
  # 查询指定表的行格式
  show table status like '表名' \G
  
  # 创建表指定行格式
  CREATE TABLE 表名 (列的信息) ROW_FORMAT=行格式名称
  
  # 修改表的行格式
  ALTER TABLE 表名 ROW_FORMAT=行格式名称
  ```

* 以下内容使用**COMPACT**行格式进行详细讲解。

## COMPACT

* 变长字段列表：把记录中所有变长字段（VARCHAR、VARBINARY、TEXT，BLOB）的真实数据占用的字节长度都存放在记录的开头部位，从而形成一个变长字段长度列表。长度值按照列的逆序存放，使用16进制进行表示。

  | 列名 | 存储内容   | 内容长度（十进制表示） | 内容长度（十六进制表示） |
  | ---- | ---------- | ---------------------- | ------------------------ |
  | col1 | 'zhangsan' | 8                      | 0x08                     |
  | col2 | 'lisi'     | 4                      | 0x04                     |
  | col3 | 'songhk'   | 6                      | 0x06                     |

  <img src="https://hhc-typora.oss-cn-shenzhen.aliyuncs.com/%E5%8F%98%E9%95%BF%E5%AD%97%E6%AE%B5%E5%88%97%E8%A1%A8.svg" alt="变长字段列表" style="zoom:125%;" />

* NULL值列表：把记录中所有为NULL值的列进行统一管理；值为1时代表列值为NULL，值为0是代表列值不为NULL。之所以要存储NULL是因为数据都是需要对齐的，如果**没有标注出来NULL值**的位置，就有可能在查询数据的时候出现**混乱**。如果使用一个特定的符号放到相应的数据位表示空置的话，虽然能达到效果，但是这样很浪费空间，所以直接就在行数据得头部开辟出一块空间专门用来记录该行数据哪些是非空数据，哪些是空数据。

  <img src="https://hhc-typora.oss-cn-shenzhen.aliyuncs.com/NULL%E5%80%BC%E5%88%97%E8%A1%A8.svg" alt="NULL值列表" style="zoom:125%;" />

* 记录头信息（5字节）

  <img src="https://hhc-typora.oss-cn-shenzhen.aliyuncs.com/%E8%AE%B0%E5%BD%95%E5%A4%B4%E4%BF%A1%E6%81%AF2.svg" alt="记录头信息2" style="zoom:150%;" />

  ---

  <img src="https://hhc-typora.oss-cn-shenzhen.aliyuncs.com/%E8%A1%8C%E6%A0%BC%E5%BC%8F%E7%AE%80%E5%8C%96%E5%9B%BE.svg" alt="行格式简化图" style="zoom:125%;" />					
  
  | 名称         | 大小（单位：bit） | 描述                                                         |
  | ------------ | ----------------- | ------------------------------------------------------------ |
  | 预留位1      | 1                 | 未使用                                                       |
  | 预留位2      | 1                 | 未使用                                                       |
  | delete_mask  | 1                 | 标记着当前记录是否被删除；0未删除，1已删除。                 |
  | min_rec_mask | 1                 | 标记当前记录是否是B+Tree非叶子节点层中最小的记录；0否，1是。 |
  | record_type  | 3                 | 当前记录的类型； 0：普通记录、1：B+树非叶节点记录、2：最小记录、3：最大记录。 |
  | heap_no      | 13                | 当前记录在页中的位置；最小记录和最大记录的heap_no值分别是0和1，详细参考**页的内部结构的Infimum + Supremum**。 |
  | n_owned      | 4                 | 标记页目录当前组中共有多少条记录，每组最后一条记录才会存储此字段；详情见页目录（page directory）。 |
  | next_record  | 16                | 表示从当前记录到下一条记录的**地址偏移量**                   |
  
  > 被删除的记录为什么还在页中存储呢？
  > 你以为它删除了，可它还在真实的磁盘上。这些被删除的记录之所以不立即从磁盘上移除，是因为移除它们之后其他的记录在磁盘上需要**重新排列，导致性能消耗**。所以只是打一个删除标记而已，所有被删除掉的记录都会组成一个所谓的**垃圾链表**，在这个链表中的记录占用的空间称之为**可重用空间**，之后如果有新记录插入到表中的话，可能把这些被删除的记录占用的存储空间覆盖掉。

  ---

  <img src="https://hhc-typora.oss-cn-shenzhen.aliyuncs.com/%E5%88%A0%E9%99%A4%E8%AE%B0%E5%BD%95%E5%89%8D.svg" alt="删除记录前" style="zoom: 150%;" />

  ---

  **删除记录**
  
  <img src="https://hhc-typora.oss-cn-shenzhen.aliyuncs.com/%E5%88%A0%E9%99%A4%E8%AE%B0%E5%BD%95%E5%90%8E.svg" alt="删除记录后" style="zoom: 150%;" />
  
  ----
  
  **添加新记录，复用被删除记录的存储空间**
  
  <img src="https://hhc-typora.oss-cn-shenzhen.aliyuncs.com/%E6%B7%BB%E5%8A%A0%E6%96%B0%E8%AE%B0%E5%BD%95.svg" alt="添加新记录" style="zoom: 150%;" />

## 行溢出

* InnoDB存储引擎可以将一条记录中的某些数据存储在真正的**数据页面之外**。

* MySQL数据库提供的VARCHAR(M)类型，真实数据是否可以存放65535个字节？

  答：否。因为除了真实的数据之外，还需要存储额外的信息，变长字段标识需要2字节，NULL值标识需要1字节。

* 页的大小一般为16KB，也就是16384个字节，而ARCHAR(M)类型的列最大可存储665533个字节，这样会出现一个页存放不了一条记录，这种现象称为**行溢出**。

* 在Compact和Reduntant行格式中，对于占用存储空间非常大的列，在记录的真实数据处只会存储该列的一部分数据，把剩余的数据分散存储在几个其他的页中进行分页存储，然后记录的真实数据处用20个字节存储指向这些页的地址（包括这些分散在其他页中数据占用的字节数），从而可以找到剩余数据所在的页。

  <img src="https://hhc-typora.oss-cn-shenzhen.aliyuncs.com/Compact%E8%A1%8C%E6%BA%A2%E5%87%BA.svg" alt="Compact行溢出" style="zoom:125%;" />

## Dynamic、Compressed

* Dynamic：动态行格式；MySQL5.7、8.0默认行格式。

* Dynamic、Compressed行格式和Compact行格式挺像，只不过在处理行溢出数据时有分歧：

  * Compressed和Dynamic两种记录格式对于存放的数据采用的是**完全行溢出**的方式。

    <img src="https://hhc-typora.oss-cn-shenzhen.aliyuncs.com/Dynamic%E8%A1%8C%E6%BA%A2%E5%87%BA.svg" alt="Dynamic行溢出" style="zoom:125%;" />

  * Compact和Redundant两种格式会在记录的真实数据处存储一部分数据（存放768个前缀字节）。

  * Compressed行记录格式的另一个功能就是，存储在其中的行数据会以zlib的算法进行压缩，因此对于BLOB、TEXT、VARCHAR这类大长度类型的数据能够进行非常有效的存储。

## Redundant

* Redundant是MySQL 5.0版本之前InnoDB的行记录存储方式，MySQL 5.0支持Redundant是为了兼容之前版本的页格式。

* 不同于Compact行记录格式，Redundant行格式的首部是一个字段长度偏移列表，同样是按照列的顺序逆序放置的。

  <img src="https://hhc-typora.oss-cn-shenzhen.aliyuncs.com/Redundant%E8%A1%8C%E6%A0%BC%E5%BC%8F%E7%A4%BA%E6%84%8F%E5%9B%BE.svg" alt="Redundant行格式示意图" style="zoom:125%;" />

* Redundant行格式会把该条记录中所有列（包括隐藏列）的长度信息都按照逆序存储到字段长度偏移列表。

* 计算列值长度的方式不像Compact行格式那么直观，它是采用两个相邻数值的差值来计算各个列值的长度，因为存储的是字段长度的偏移量。

* 不同于Compact行格式，Redundant行格式中的记录头信息固定占用6个字节（48位）。

  | 名称            | 大小（bit） | 描述                                                         |
  | --------------- | ----------- | ------------------------------------------------------------ |
  | ()              | 1           | 末使用                                                       |
  | ()              | 1           | 末使用                                                       |
  | deleted_mask    | 1           | 该行是否已被删除                                             |
  | min_rec_mask    | 1           | B+树的每层非叶子节点中的最小记录都会添加该标记               |
  | n_owned         | 4           | 标记页目录当前组中共有多少条记录                             |
  | heap_no         | 13          | 素引堆中该条记录的位置信息                                   |
  | n_fields        | 10          | 记录中列的数量                                               |
  | 1byte_offs_flag | 1           | 记录字段长度偏移列表中每个列对应的偏移量，使用1个字节还是2个字节表示 |
  | next_record     | 16          | 页中下一条记录的绝对位置                                     |
  

# 表空间

* 表空间可以看做是InnoDB存储引擎逻辑结构的最高层，所有的数据都存放在表空间中。

* 表空间是一个**逻辑容器**，表空间存储的对象是段，在一个表空间中可以有一个或多个段，但是一个段只能属于一个表空间。

* 表空间从管理上可划分为：系统表空间（System tablespace）、独立表空间（File-per-table tablespace）、撤销表空间（Undo Tablespace）和临时表空间（Temporary Tablespace）等。

* 独立表空间：

  * 每张表有一个独立的表空间，也就是数据和索引信息都会保存在自己的表空间中。独立的表空间（即：单表）可以在不同的数据库之间进**迁移**。
  * 空间可以回收（DROP TABLE操作可自动回收表空间；其他情况，表空间不能自己回收）。如果对于统计分析或是日志表，删除大量数据后可以通过：alter table 表名 engine=innodb 回收不用的空间。对于使用独立表空间的表，不管怎么删除，表空间的碎片不会太严重的影响性能，而且还有机会处理。
  * 新建的表对应的`.ibd`文件只占用96K，才6个页面大小（MySQL5.7中），这是因为一开始表空间占用的空间很小，因为表里边都没有数据。不过别忘了这些.ibd文件是自扩展的，随着表中数据的增多，表空间对应的文件也逐渐增大。MySQL8.0中是 7个页面大小。原因`.idb`还存储了表结构，表结构`.frm`文件在MySQL8.0中已被移除。

* 系统表空间：

  * 系统表空间的结构和独立表空间基本类似，只不过由于整个MySQL进程只有一个系统表空间，在系统表空间中会额外记录一些有关整个系统信息的页面，这部分是独立表空间中没有的。

  * MySQL除了保存着我们插入的用户数据之外，还需要保存许多额外的信息。

    | 表名             | 描述                                                         |
    | ---------------- | ------------------------------------------------------------ |
    | SYS_TABLES       | 整个InnoDB存储引擎中所有的表的信息                           |
    | SYS_COLUMNS      | 整个InnoDB存储引擎中所有的列的信息                           |
    | SYS_INDEXES      | 整个InnoDB存储引擎中所有的索引的信息                         |
    | SYS_FIELDS       | 整个InnoDB存储引擎中所有的索引对应的列的信息                 |
    | SYS_FOREIGN      | 整个InnoDB存储引擎中所有的外键的信息                         |
    | SYS_FOREIGN_COLS | 整个InnoDB存储引擎中所有的外键对应列的信息                   |
    | SYS_TABLESPACES  | 整个InnoDB存储引擎中所有的表空间信息                         |
    | SYS_DATAFILES    | 整个 InnoDB 存储引擎中所有的表空间对应文件系统的文件路径信息 |
    | SYS_VIRTUAL      | 整个InnoDB存储引擎中所有的虚拟生成列的信息                   |

    > 这些表都存储在系统表空间中。
    >
    > 注意：用户是无法直接访问这些内部系统表的，不过考虑到查看这些表的内容可能有助于大家分析问题，所以在系统数据库**information_schema**中提供了一些以**innodb_sys**开头的表。
    >
    > ```mysql
    > USE information_schema;
    > SHOW TABLES LIKE 'innodb_sys%';
    > ```

# 扩展

* 数据页加载的三种方式：内存读取、随机读取、顺序读取。