##综述

从上层的角度来看，InnoDB层的文件，除了redo日志外，基本上具有相当统一的结构，都是固定block大小，普遍使用的btree结构来管理数据。只是针对不同的block的应用场景会分配不同的页类型。通常默认情况下，每个block的大小为UNIV_PAGE_SIZE，在不做任何配置时值为16kb，你还可以选择在安装实例时指定一个块的block大小。 对于压缩表，可以在建表时指定block size，但在内存中表现的解压页依旧为统一的页大小。

从物理文件的分类来看，有日志文件，主系统表空间文件ibdata，undo tablespace文件，临时表空间文件，用户表空间。

日志文件主要用于记录redo log，InnoDB采用循环使用的方式，你可以通过参数指定创建文件的个数和每个文件的大小。默认情况下，日志是以512字节的block单位写入。由于现代文件系统的block size通常设置到4k，InnoDB提供了一个选项，可以让用户将写入的redo日志填充到4KB，以避免read-modify-write的现象；而Percona Server则提供了另外一个选项，支持直接将redo日志的block size修改成指定的值。

ibdata是InnoDB最重要的系统表空间文件，它记录了InnoDB的核心信息，包括事务系统信息，元数据信息，记录InnoDB change buffer的btree， 防止数据损坏的double write buffer等等关键信息。我们稍后会展开描述。

undo独立表空间是一个可选项，通常默认情况下，undo数据是存储在ibdata中的，但你也可以通过配置选项innodb_undo_tablespaces来将undo 回滚段分配到不同的文件中，目前开启undo tablespace只能在install阶段进行。在主流版本进入5.7时代后，我们建议开启独立undo表空间，只有这样才能利用到5.7引入的新特效：online undo truncate。

MySQL 5.7新开辟了一个临时表空间，默认的磁盘文件命名为ibtmp1，所有非压缩的临时表都存储在该表空间中。由于临时表的本身属性，该文件在重启时会重新创建。对于云服务提供商而言，通过ibtmp文件，可以更好的控制临时文件产生的磁盘存储。

用户表空间，顾名思义，就是用于自己创建的表空间，通常分为两类，一类是一个表空间一个文件，另外一种则是5.7版本引入的所谓General Tablespace，在满足一定约束条件下，可以将多个表创建到同一个文件中。除此之外，InnoDB还定义了一些特殊用途的ibd文件，例如全文索引相关的表文件。而针对空间数据类型，也构建了不同的数据索引格式R-tree。

为了管理磁盘文件的读写操作，InnoDB设计了一套文件IO操作接口，提供了同步IO和异步IO两种文件读写方式。针对异步IO，支持两种方式：一种是Native AIO，这需要你在编译阶段加上LibAio的Dev包，另外一种是simulated aio模式，InnoDB早期实现了一套系统来模拟异步IO，但现在Native Aio已经很成熟了，并且Simulated Aio本身存在性能问题，建议生产环境开启Native Aio模式。

对于数据读操作，通常用户线程触发的数据块请求读是同步读，如果开启了数据预读机制的话，预读的数据块则为异步读，由后台IO线程进行。其他后台线程也会触发数据读操作，例如Purge线程在无效数据清理，会读undo页和数据页；Master线程定期做ibuf merge也会读入数据页。崩溃恢复阶段也可能触发异步读来加速recover的速度。

对于数据写操作，InnoDB和大部分数据库系统一样，都是WAL模式，即先写日志，延迟写数据页。事务日志的写入通常在事务提交时触发，后台master线程也会每秒做一次redo fsync。数据页则通常由后台Page cleaner线程触发。但当buffer pool空闲block不够时，或者没做checkpoint的lsn age太长时，也会驱动刷脏操作，这两种场景由用户线程来触发。Percona Server据此做了优化来避免用户线程参与。MySQL5.7也对应做了些不一样的优化。

除了数据块操作，还是物理文件级别的操作，例如truncate, drop table，rename table等DDL操作，InnoDB需要对这些操作进行协调，目前的解法是通过特殊的flag和计数器的方式来解决。

当文件读入内存后，我们需要一种统一的方式来对数据进行管理，在启动实例时，InnoDB会按照instance分区分配多个一大块内存（在5.7里则是按照可配置的chunk size进行内存块划分），每个chunk又以UNIV_PAGE_SIZE为单位进行划分。数据读入内存时，会从buffer pool的free list中分配一个空闲block。所有的数据页都存储在一个LRU链表上。修改过的block被加到flush_list上，解压的数据页被放到unzip_LRU链表上。我们可以配置buffer pool为多个instance，以降低对链表的竞争开销。

从物理文件到内存管理是一个相对比较庞大的架构，本文将一一为读者进行分析解读，以让读者对InnoDB的文件系统管理有个更加全面的认识。在关键的地方本文注明了代码函数，建议读者边参考代码边阅读本文。

本文的代码部分基于MySQL 5.7.11版本，不同的版本函数名或逻辑可能会有所不同。请读者阅读本文时尽量选择该版本的代码。

##物理文件

本小节主要从文件的物理结构的角度阐述InnoDB在最底层如何对物理文件进行管理，再分别介绍各类文件的不同结构。

###文件管理页

InnoDB的每个数据文件都归属于一个表空间，不同的表空间使用一个唯一标识的space id来标记。例如ibdata1, ibdata2...归属系统表空间，拥有相同的space id。用户创建表产生的ibd文件，则认为是一个独立的tablespace，只包含一个文件。

每个文件按照固定的page size进行区分，默认情况下，非压缩表的page size为16Kb。而在文件内部又按照64个Page(总共1M)一个Extent的方式进行划分并管理。对于不同的page size，对应的Extent大小也不同，对应为：

| page size | file space extent size
------------|-----------
| 4 KiB  | 256 pages = 1 MiB
| 8 KiB  | 128 pages = 1 MiB
| 16 KiB  | 64 pages = 1 MiB
| 32 KiB   | 64 pages = 2 MiB
| 64 KiB  | 64 pages = 4 MiB
  
尽管支持更大的Page Size，但目前还不支持大页场景下的数据压缩，原因是这涉及到修改压缩页中slot的固定size（其实实现起来也不复杂）。在不做声明的情况下，下文我们默认使用16KB的Page Size来阐述文件的物理结构。

为了管理整个Tablespace，除了索引页外，数据文件中还包含了多种管理页，如下图所示，一个用户表空间大约包含这些页来管理文件，下面会一一进行介绍。

![1](http://mysql.taobao.org/monthly/pic/2016-02-01/1.png)


####文件链表

首先我们先介绍基于文件的一个基础结构，即文件链表。为了管理Page，Extent这些数据块，在文件中记录了许多的节点以维持具有某些特征的链表，例如在在文件头维护的inode page链表，空闲、用满以及碎片化的Extent链表等等。

在InnoDB里链表头称为FLST_BASE_NODE，大小为FLST_BASE_NODE_SIZE(16个字节)。BASE NODE维护了链表的头指针和末尾指针，每个节点称为FLST_NODE，大小为FLST_NODE_SIZE（12个字节）。相关结构描述如下：

FLST_BASE_NODE:

| Macro | bytes | Desc
--------------- |---------| ------
| FLST_LEN | 4 | 存储链表的长度
| FLST_FIRST| 6 | 指向链表的第一个节点
| FLST_LAST | 6 | 指向链表的最后一个节点

FLST_NODE:

| Macro | bytes | Desc
--------------- |---------| ------
| FLST_PREV | 6 | 指向当前节点的前一个节点
| FLST_NEXT | 6 | 指向当前节点的下一个节点

如上所述，文件链表中使用6个字节来作为节点指针，指针的内容包括：

| Macro | bytes | Desc
--------------- |---------| ------
| FIL_ADDR_PAGE | 4 | Page No
| FIL_ADDR_BYTE | 2 | Page内的偏移量

该链表结构是InnoDB表空间内管理所有page的基础结构，下图先感受下，具体的内容可以继续往下阅读。

![2](http://mysql.taobao.org/monthly/pic/2016-02-01/2.png)

文件链表管理的相关代码参阅：include/fut0lst.ic, fut/fut0lst.cc

#### FSP_HDR PAGE

数据文件的第一个Page类型为FIL_PAGE_TYPE_FSP_HDR，在创建一个新的表空间时进行初始化(`fsp_header_init`)，该page同时用于跟踪随后的256个Extent(约256MB文件大小)的空间管理，所以每隔256MB就要创建一个类似的数据页，类型为FIL_PAGE_TYPE_XDES ，XDES Page除了文件头部外，其他都和FSP_HDR页具有相同的数据结构，可以称之为Extent描述页，每个Extent占用40个字节，一个XDES Page最多描述256个Extent。

FSP_HDR页的头部使用FSP_HEADER_SIZE个字节来记录文件的相关信息，具体的包括：

| Macro | bytes | Desc
--------------- |---------| ------
| FSP_SPACE_ID | 4  | 该文件对应的space id
| FSP_NOT_USED | 4 | 如其名，保留字节，当前未使用
| FSP_SIZE | 4 | 当前表空间总的PAGE个数，扩展文件时需要更新该值（`fsp_try_extend_data_file_with_pages`）
| FSP_FREE_LIMIT | 4 | 当前尚未初始化的最小Page No。从该Page往后的都尚未加入到表空间的FREE LIST上。
| FSP_SPACE_FLAGS | 4 | 当前表空间的FLAG信息，见下文
| FSP_FRAG_N_USED | 4 | FSP_FREE_FRAG链表上已被使用的Page数，用于快速计算该链表上可用空闲Page数
| FSP_FREE | 16 | 当一个Extent中所有page都未被使用时，放到该链表上，可以用于随后的分配
| FSP_FREE_FRAG | 16 | FREE_FRAG链表的Base Node，通常这样的Extent中的Page可能归属于不同的segment，用于segment frag array page的分配（见下文）
| FSP_FULL_FRAG | 16 | Extent中所有的page都被使用掉时，会放到该链表上，当有Page从该Extent释放时，则移回FREE_FRAG链表
| FSP_SEG_ID | 8 | 当前文件中最大Segment ID + 1，用于段分配时的seg id计数器
| FSP_SEG_INODES_FULL | 16 | 已被完全用满的Inode Page链表
| FSP_SEG_INODES_FREE | 16 | 至少存在一个空闲Inode Entry的Inode Page被放到该链表上

在文件头使用FLAG（对应上述FSP_SPACE_FLAGS）描述了创建表时的如下关键信息：

| Macro | Desc
--------------- |---------
| FSP_FLAGS_POS_ZIP_SSIZE | 压缩页的block size，如果为0表示非压缩表
| FSP_FLAGS_POS_ATOMIC_BLOBS | 使用的是compressed或者dynamic的行格式
| FSP_FLAGS_POS_PAGE_SSIZE | Page Size
| FSP_FLAGS_POS_DATA_DIR | 如果该表空间显式指定了data_dir，则设置该flag
| FSP_FLAGS_POS_SHARED | 是否是共享的表空间，如5.7引入的General Tablespace，可以在一个表空间中创建多个表
| FSP_FLAGS_POS_TEMPORARY | 是否是临时表空间
| FSP_FLAGS_POS_ENCRYPTION | 是否是加密的表空间，MySQL 5.7.11引入
| FSP_FLAGS_POS_UNUSED | 未使用的位
 
除了上述描述信息外，其他部分的数据结构和XDES PAGE（FIL_PAGE_TYPE_XDES）都是相同的，使用连续数组的方式，每个XDES PAGE最多存储256个XDES Entry,每个Entry占用40个字节，描述64个Page（即一个Extent）。格式如下：

| Macro | bytes | Desc
--------------- |---------| ------
| XDES_ID | 8 | 如果该Extent归属某个segment的话，则记录其ID
| XDES_FLST_NODE | 12(FLST_NODE_SIZE) | 维持Extent链表的双向指针节点
| XDES_STATE | 4 | 该Extent的状态信息，包括：XDES_FREE，XDES_FREE_FRAG，XDES_FULL_FRAG，XDES_FSEG，详解见下文
| XDES_BITMAP| 16 | 总共16*8= 128个bit，用2个bit表示Extent中的一个page，一个bit表示该page是否是空闲的(XDES_FREE_BIT)，另一个保留位，尚未使用（XDES_CLEAN_BIT）

XDES_STATE表示该Extent的四种不同状态：

| Macro| Desc |
--------------- |---------
| XDES_FREE(1) | 存在于FREE链表上
| XDES_FREE_FRAG(2) | 存在于FREE_FRAG链表上
| XDES_FULL_FRAG(3) | 存在于FULL_FRAG链表上
| XDES_FSEG(4) | 该Extent归属于ID为XDES_ID记录的值的SEGMENT。

通过XDES_STATE信息，我们只需要一个FLIST_NODE节点就可以维护每个Extent的信息，是处于全局表空间的链表上，还是某个btree segment的链表上。

####IBUF BITMAP PAGE

第2个page类型为FIL_PAGE_IBUF_BITMAP，主要用于跟踪随后的每个page的change buffer信息，使用4个bit来描述每个page的change buffer信息。

| Macro | bits | Desc
--------------- |---------| ------
| IBUF_BITMAP_FREE | 2 | 使用2个bit来描述page的空闲空间范围：0（0 bytes）、1（512 bytes）、2（1024 bytes）、3（2048 bytes）
| IBUF_BITMAP_BUFFERED | 1 | 是否有ibuf操作缓存
| IBUF_BITMAP_IBUF | 1 | 该Page本身是否是Ibuf Btree的节点

由于bitmap page的空间有限，同样每隔256个Extent Page之后，也会在XDES PAGE之后创建一个ibuf bitmap page。

关于change buffer，这里我们不展开讨论，感兴趣的可以阅读之前的这篇月报：
[MySQL · 引擎特性 · Innodb change buffer介绍](http://mysql.taobao.org/monthly/2015/07/01/)

####INODE PAGE

数据文件的第3个page的类型为FIL_PAGE_INODE，用于管理数据文件中的segement，每个索引占用2个segment，分别用于管理叶子节点和非叶子节点。每个inode页可以存储FSP_SEG_INODES_PER_PAGE（默认为85）个记录。

| Macro | bits | Desc
--------------- |---------| ------
| FSEG_INODE_PAGE_NODE | 12 |  INODE页的链表节点，记录前后Inode Page的位置，BaseNode记录在头Page的FSP_SEG_INODES_FULL或者FSP_SEG_INODES_FREE字段。
| Inode Entry 0 | 192 | Inode记录
| Inode Entry 1 | |
| …… | |
| Inode Entry 84 || 

每个Inode Entry的结构如下表所示：

| Macro | bits | Desc
--------------- |---------| ------
| FSEG_ID | 8 | 该Inode归属的Segment ID，若值为0表示该slot未被使用
| FSEG_NOT_FULL_N_USED| 8 | FSEG_NOT_FULL链表上被使用的Page数量
| FSEG_FREE | 16 | 完全没有被使用并分配给该Segment的Extent链表
| FSEG_NOT_FULL | 16 | 至少有一个page分配给当前Segment的Extent链表，全部用完时，转移到FSEG_FULL上，全部释放时，则归还给当前表空间FSP_FREE链表
| FSEG_FULL | 16 |  分配给当前segment且Page完全使用完的Extent链表
| FSEG_MAGIC_N | 4 | Magic Number
| FSEG_FRAG_ARR 0 | 4 | 属于该Segment的独立Page。总是先从全局分配独立的Page，当填满32个数组项时，就在每次分配时都分配一个完整的Extent，并在XDES PAGE中将其Segment ID设置为当前值
| …… | …… | 
| FSEG_FRAG_ARR 31 | 4| 总共存储32个记录项

####文件维护

从上文我们可以看到，InnoDB通过Inode Entry来管理每个Segment占用的数据页，每个segment可以看做一个文件页维护单元。Inode Entry所在的inode page有可能存放满，因此又通过头Page维护了Inode Page链表。

在ibd的第一个Page中还维护了表空间内Extent的FREE、FREE_FRAG、FULL_FRAG三个Extent链表；而每个Inode Entry也维护了对应的FREE、NOT_FULL、FULL三个Extent链表。这些链表之间存在着转换关系，以高效的利用数据文件空间。

当创建一个新的索引时，实际上构建一个新的btree(`btr_create`)，先为非叶子节点Segment分配一个inode entry，再创建root page，并将该segment的位置记录到root page中，然后再分配leaf segment的Inode entry，并记录到root page中。

当删除某个索引后，该索引占用的空间需要能被重新利用起来。

**创建Segment**

首先每个Segment需要从ibd文件中预留一定的空间(`fsp_reserve_free_extents`)，通常是2个Extent。但如果是新创建的表空间，且当前的文件小于1个Extent时，则只分配2个Page。

当文件空间不足时，需要对文件进行扩展(`fsp_try_extend_data_file`)。文件的扩展遵循一定的规则：如果当前小于1个Extent，则扩展到1个Extent满；当表空间小于32MB时，每次扩展一个Extent；大于32MB时，每次扩展4个Extent（`fsp_get_pages_to_extend_ibd`）。

在预留空间后，读取文件头Page并加锁（`fsp_get_space_header`），然后开始为其分配Inode Entry(`fsp_alloc_seg_inode`)。 首先需要找到一个合适的inode page。

我们知道Inode Page的空间有限，为了管理Inode Page，在文件头存储了两个Inode Page链表，一个链接已经用满的inode page，一个链接尚未用满的inode page。如果当前Inode Page的空间使用完了，就需要再分配一个inode page，并加入到FSP_SEG_INODES_FREE链表上(`fsp_alloc_seg_inode_page`)。对于独立表空间，通常一个inode page就足够了。

当拿到目标inode page后，从该Page中找到一个空闲（`fsp_seg_inode_page_find_free`）未使用的slot（空闲表示其不归属任何segment，即FSEG_ID置为0）

一旦该inode page中的记录用满了，就从FSP_SEG_INODES_FREE链表上转移到FSP_SEG_INODES_FULL链表。

获得inode entry后，递增头page的FSP_SEG_ID，作为当前segment的seg id写入到inode entry中。随后进行一些列的初始化。

在完成inode entry的提取后，就将该inode entry所在inode page的位置及页内偏移量存储到其他某个page内（对于btree就是记录在根节点内，占用10个字节，包含space id, page no, offset）。

Btree的根节点实际上是在创建non-leaf segment时分配的，root page被分配到该segment的frag array的第一个数组元素中。

Segment分配入口函数： `fseg_create_general`

**分配数据页**

随着btree数据的增长，我们需要为btree的segment分配新的page。前面我们已经讲过，segment是一个独立的page管理单元，我们需要将从全局获得的数据空间纳入到segment的管理中。

Step 1： 空间扩展

当判定插入索引的操作可能引起分裂时，会进行悲观插入(`btr_cur_pessimistic_insert`)，在做实际的分裂操作之前，会先对文件进行扩展，并尝试预留(tree_height / 16 + 3)个Extent，大多数情况下都是3个Extent。

这里有个意外场景：如果当前文件还不超过一个Extent，并且请求的page数小于1/2个Extent时，则如果指定page数，保证有2个可用的空闲Page，或者分配指定的page，而不是以Extent为单位进行分配。

注意这里只是保证有足够的文件空间，避免在btree操作时进行文件Extent。如果在这一步扩展了ibd文件(`fsp_try_extend_data_file`)，新的数据页并未初始化，也未加入到任何的链表中。

在判定是否有足够的空闲Extent时，本身ibd预留的空闲空间也要纳入考虑，对于普通用户表空间是2个Extent + file_size * 1%。 这些新扩展的page此时并未进行初始化，也未加入到，在头page的FSP_FREE_LIMIT记录的page no标识了这类未初始化页的范围。

Step 2：为segment分配page

随后进入索引分裂阶段(`btr_page_split_and_insert`)，新page分配的上层调用栈：

```
btr_page_alloc
|--> btr_page_alloc_low 
	|--> fseg_alloc_free_page_general
		|--> fseg_alloc_free_page_low
```

在传递的参数中，有个hint page no，通常是当前需要分裂的page no的前一个（direction = FSP_DOWN）或者后一个page no(direction = FSP_UP)，其目的是将逻辑上相邻的节点在物理上也尽量相邻。
	
在Step 1我们已经保证了物理空间有足够的数据页，只是还没进行初始化。将page分配到当前segment的流程如下(`fseg_alloc_free_page_low`)：

* 计算当前segment使用的和占用的page数
    * 使用的page数存储包括FSEG_NOT_FULL链表上使用的page数(存储在inode entry的FSEG_NOT_FULL_N_USED中) + 已用满segment的FSEG_FULL链表上page数 + 占用的frag array page数量
    * 占用的page数包括FSEG_FREE、FSEG_NOT_FULL 、FSEG_FULL三个链表上的Extent + 占用的frag array page数量。
* 根据hint page获取对应的xdes entry (`xdes_get_descriptor_with_space_hdr`)
* 当满足如下条件时该hint page可以直接拿走使用：
    * Extent状态为XDES_FSEG，表示属于一个segment
    * hint page所在的Extent已被分配给当前segment(检查xdes entry的XDES_ID)
    * hint page对应的bit设置为free，表示尚未被占用
    * **返回hint page**
* 当满足条件：1. xdes entry当前是空闲状态(XDES_FREE)；2.该segment中已使用的page数大于其占用的page数的7/8 (FSEG_FILLFACTOR)；3. 当前segment已经使用了超过32个frag page，即表示其inode中的frag array可能已经用满。
    * 从表空间分配hint page所在的Extent (`fsp_alloc_free_extent`)，将其从FSP_FREE链表上移除
    * 设置该Extent的状态为XDES_FSEG，写入seg id，并加入到当前segment的FSEG_FREE链表中。
    * **返回hint page**
* 当如下条件时：1. direction != FSP_NO_DIR，对于Btree分裂，要么FSP_UP，要么FSP_DOWN；2.已使用的空间小于已占用空间的7/8； 3.当前segment已经使用了超过32个frag page
    * 尝试从segment获取一个Extent(`fseg_alloc_free_extent`)，如果该segment的FSEG_FREE链表为空，则需要从表空间分配（`fsp_alloc_free_extent`）一个Extent，并加入到当前segment的FSEG_FREE链表上
    * direction为FSP_DOWN时，**返回该Extent最后一个page**，为FSP_UP时**返回该Extent的第一个Page**
* xdes entry属于当前segment且未被用满，从其中取一个**空闲page并返回**
* 如果该segment占用的page数大于实用的page数，说明该segment还有空闲的page，则依次先看FSEG_NOT_FULL链表上是否有未满的Extent，如果没有，再看FSEG_FREE链表上是否有完全空闲的Extent。从其中取一个**空闲Page并返回**
* 当前已经实用的Page数小于32个page时，则分配独立的page（`fsp_alloc_free_page`）并加入到该inode的frag array page数组中，然后**返回该block**
* 当上述情况都不满足时，直接分配一个Extent(`fseg_alloc_free_extent`)，并从**其中取一个page返回**。

上述流程看起来比较复杂，但可以总结为：
1. 对于一个新的segment，总是优先填满32个frag page数组，之后才会为其分配完整的Extent，可以利用碎片页，并避免小表占用太多空间。
2. 尽量获得hint page;
3. 如果segment上未使用的page太多，则尽量利用segment上的page。

上文提到两处从表空间为segment分配数据页，一个是分配单独的数据页，一个是分配整个Extent

表空间单独数据页的分配调用函数`fsp_alloc_free_page`:

* 如果hint page所在的Extent在链表XDES_FREE_FRAG上，可以直接使用；否则从根据头page的FSP_FREE_FRAG链表查看是否有可用的Extent；
* 未能从上述找到一个可用Extent，直接分配一个Extent，并加入到FSP_FREE_FRAG链表中。
* 从获得的Extent中找到描述为空闲（XDES_FREE_BIT）的page。
* 分配该page (`fsp_alloc_from_free_frag`)
    * 设置page对应的bitmap的XDES_FREE_BIT为false,表示被占用 
    * 递增头page的FSP_FRAG_N_USED字段
    * 如果该Extent被用满了，就将其从FSP_FREE_FRAG移除，并加入到FSP_FULL_FRAG链表中。同时对头Page的FSP_FRAG_N_USED递减1个Extent(FSP_FRAG_N_USED只存储未满的Extent使用的page数量)。
    * 对Page内容进行初始化(`fsp_page_create`)

表空间Extent的分配函数`fsp_alloc_free_extent`: 

* 通常先通过头page看FSP_FREE链表上是否有空闲的Extent，如果没有的话，则将新的Extent（例如上述step 1对文件做扩展产生的新page，从FSP_FREE_LIMIT算起）加入到FSP_FREE链表上(`fsp_fill_free_list`)：
     * 一次最多加4个Extent(`FSP_FREE_ADD`)
     * 如果涉及到xdes page，还需要对xdes page进行初始化；
     * 如果Extent中存在类似xdes page这样的系统管理页，这个Extent被加入到FSP_FREE_FRAG链表中而不是FSP_FREE链表。
     * 取链表上第一个Extent为当前使用。
* 将获得的Extent从FSP_FREE移除，并返回对应的xdes entry(`xdes_lst_get_descriptor`)

**回收Page**

数据页的回收分为两种，一种是整个Extent的回收，一种是碎片页的回收。在删除索引页或者drop索引时都会发生。

当某个数据页上的数据被删光时，我们需要从其所在segmeng上删除该page（`btr_page_free -->fseg_free_page --> fseg_free_page_low`），回收的流程也比较简单：

* 首先如果是该segment的frag array中的page，将对应的slot设置为FIL_NULL, 并返还给表空间(`fsp_free_page`):
    * page在xdes entry中的状态置为空闲
    * 如果page所在Extent处于FSP_FULL_FRAG链表，则转移到FSP_FREE_FRAG中
    * 如果Extent中的page完全被释放掉了，则释放该Extent(`fsp_free_extent`)，将其转移到FSP_FREE链表
    * 从函数**返回**
* 如果page所处于的Extent当前在该segment的FSEG_FULL链表上，则转移到FSEG_NOT_FULL链表
* 设置Page在xdes entry的bitmap对应的XDES_FREE_BIT为true
* 如果此时该Extent上的page全部被释放了，将其从FSEG_NOT_FULL链表上移除，并加入到表空间的FSP_FREE链表上(而非Segment的FSEG_FREE链表)。

**释放Segment**

当我们删除索引或者表时，需要删除btree（`btr_free_if_exists`），先删除除了root节点外的其他部分(`btr_free_but_not_root`)，再删除root节点(`btr_free_root`)

由于数据操作都需要记录redo，为了避免产生非常大的redo log，leaf segment通过反复调用函数`fseg_free_step`来释放其占用的数据页：

* 首先找到leaf segment对应的Inode entry（`fseg_inode_try_get`）
* 然后依次查找inode entry中的FSEG_FULL、或者FSEG_NOT_FULL、或者FSEG_FREE链表，找到一个Extent，注意着里的链表元组所指向的位置实际上是描述该Extent的Xdes Entry所在的位置。因此可以快速定位到对应的Xdes Page及Page内偏移量(`xdes_lst_get_descriptor`)
* 现在我们可以将这个Extent安全的释放了(`fseg_free_extent`，见后文)
* 当反复调用fseg_free_step将所有的Extent都释放后，segment还会最多占用32个碎片页，也需要依次释放掉(`fseg_free_page_low`)
* 最后，当该inode所占用的page全部释放时，释放inode entry：
    * 如果该inode所在的inode page中当前被用满，则由于我们即将释放一个slot，需要从FSP_SEG_INODES_FULL转移到FSP_SEG_INODES_FREE（更新第一个page）
    * 将该inode entry的SEG_ID清除为0，表示未使用
    * 如果该inode page上全部inode entry都释放了，就从FSP_SEG_INODES_FREE移除，并删除该page。

non-leaf segment的回收和leaf segment的回收基本类似，但要注意btree的根节点存储在该segment的frag arrary的第一个元组中，该Page暂时不可以释放(`fseg_free_step_not_header`)

btree的root page在完成上述步骤后再释放，此时才能彻底释放non-leaf segment

###索引页

ibd文件中真正构建起用户数据的结构是BTREE，在你创建一个表时，已经基于显式或隐式定义的主键构建了一个btree，其叶子节点上记录了行的全部列数据（加上事务id列及回滚段指针列）；如果你在表上创建了二级索引，其叶子节点存储了键值加上聚集索引键值。本小节我们探讨下组成索引的物理存储页结构，这里默认讨论的是非压缩页，我们在下一小节介绍压缩页的内容。

每个btree使用两个Segment来管理数据页，一个管理叶子节点，一个管理非叶子节点，每个segment在inode page中存在一个记录项，在btree的root page中记录了两个segment信息。

当我们需要打开一张表时，需要从ibdata的数据词典表中load元数据信息，其中SYS_INDEXES系统表中记录了表，索引，及索引根页对应的page no（`DICT_FLD__SYS_INDEXES__PAGE_NO`），进而找到btree根page，就可以对整个用户数据btree进行操作。

索引最基本的页类型为FIL_PAGE_INDEX。可以划分为下面几个部分。

**Page Header**

首先不管任何类型的数据页都有38个字节来描述头信息（FIL_PAGE_DATA, or PAGE_HEADER），包含如下信息：

| Macro | bytes | Desc
--------------- |---------| ------
| FIL_PAGE_SPACE_OR_CHKSUM | 4 | 在MySQL4.0之前存储space id，之后的版本用于存储checksum
| FIL_PAGE_OFFSET | 4 | 当前页的page no
| FIL_PAGE_PREV | 4 | 通常用于维护btree同一level的双向链表，指向链表的前一个page，没有的话则值为FIL_NULL
| FIL_PAGE_NEXT | 4 | 和FIL_PAGE_PREV类似，记录链表的下一个Page的Page No
| FIL_PAGE_LSN | 8 | 最近一次修改该page的LSN
| FIL_PAGE_TYPE | 2 | Page类型
| FIL_PAGE_FILE_FLUSH_LSN | 8 | 只用于系统表空间的第一个Page，记录在正常shutdown时安全checkpoint到的点，对于用户表空间，这个字段通常是空闲的，但在5.7里，FIL_PAGE_COMPRESSED类型的数据页则另有用途。下一小节单独介绍
| FIL_PAGE_SPACE_ID | 4 | 存储page所在的space id

**Index Header**

紧随FIL_PAGE_DATA之后的是索引信息，这部分信息是索引页独有的。

| Macro | bytes | Desc
--------------- |---------| ------
| PAGE_N_DIR_SLOTS | 2 | Page directory中的slot个数 （见下文关于Page directory的描述）
| PAGE_HEAP_TOP | 2 | 指向当前Page内已使用的空间的末尾便宜位置，即free space的开始位置
| PAGE_N_HEAP | 2 | Page内所有记录个数，包含用户记录，系统记录以及标记删除的记录，同时当第一个bit设置为1时，表示这个page内是以Compact格式存储的
| PAGE_FREE | 2 | 指向标记删除的记录链表的第一个记录
| PAGE_GARBAGE | 2 | 被删除的记录链表上占用的总的字节数，属于可回收的垃圾碎片空间
| PAGE_LAST_INSERT | 2 | 指向最近一次插入的记录偏移量，主要用于优化顺序插入操作
| PAGE_DIRECTION | 2 | 用于指示当前记录的插入顺序以及是否正在进行顺序插入，每次插入时，PAGE_LAST_INSERT会和当前记录进行比较，以确认插入方向，据此进行插入优化
| PAGE_N_DIRECTION | 2 | 当前以相同方向的顺序插入记录个数
| PAGE_N_RECS | 2 | Page上有效的未被标记删除的用户记录个数
| PAGE_MAX_TRX_ID | 8 | 最近一次修改该page记录的事务ID，主要用于辅助判断二级索引记录的可见性。
| PAGE_LEVEL | 2 | 该Page所在的btree level，根节点的level最大，叶子节点的level为0
| PAGE_INDEX_ID | 8 | 该Page归属的索引ID


**Segment Info**

随后20个字节描述段信息，仅在Btree的root Page中被设置，其他Page都是未使用的。

| Macro | bytes | Desc
--------------- |---------| ------
| PAGE_BTR_SEG_LEAF | 10(FSEG_HEADER_SIZE) | leaf segment在inode page中的位置
| PAGE_BTR_SEG_TOP | 10(FSEG_HEADER_SIZE) | non-leaf segment在inode page中的位置

10个字节的inode信息包括：

| Macro | bytes | Desc
--------------- |---------| ------
| FSEG_HDR_SPACE | 4 | 描述该segment的inode page所在的space id （目前的实现来看，感觉有点多余...）
| FSEG_HDR_PAGE_NO | 4 | 描述该segment的inode page的page no 
| FSEG_HDR_OFFSET | 2 | inode page内的页内偏移量

通过上述信息，我们可以找到对应segment在inode page中的描述项，进而可以操作整个segment。

**系统记录**

之后是两个系统记录，分别用于描述该page上的极小值和极大值，这里存在两种存储方式，分别对应旧的InnoDB文件系统，及新的文件系统（compact page）

| Macro | bytes | Desc
--------------- |---------| ------
| REC_N_OLD_EXTRA_BYTES + 1 | 7 |  固定值，见infimum_supremum_redundant的注释
| PAGE_OLD_INFIMUM | 8 | "infimum\0"
| REC_N_OLD_EXTRA_BYTES + 1 | 7 | 固定值，见infimum_supremum_redundant的注释
| PAGE_OLD_SUPREMUM | 9 | "supremum\0"

Compact的系统记录存储方式为：

| Macro | bytes | Desc
--------------- |---------| ------
| REC_N_NEW_EXTRA_BYTES | 5 | 固定值，见infimum_supremum_compact的注释
| PAGE_NEW_INFIMUM | 8 | "infimum\0"
| REC_N_NEW_EXTRA_BYTES | 5 | 固定值，见infimum_supremum_compact的注释
| PAGE_NEW_SUPREMUM | 8 | "supremum"，这里不带字符0

两种格式的主要差异在于不同行存储模式下，单个记录的描述信息不同。在实际创建page时，系统记录的值已经初始化好了，对于老的格式(REDUNDANT)，对应代码里的`infimum_supremum_redundant`，对于新的格式(compact)，对应`infimum_supremum_compact`。infimum记录的固定heap no为0，supremum记录的固定Heap no 为1。page上最小的用户记录前节点总是指向infimum，page上最大的记录后节点总是指向supremum记录。

具体参考索引页创建函数：`page_create_low`

**用户记录**

在系统记录之后就是真正的用户记录了，heap no 从2（PAGE_HEAP_NO_USER_LOW）开始算起。注意Heap no仅代表物理存储顺序，不代表键值顺序。

根据不同的类型，用户记录可以是非叶子节点的Node指针信息，也可以是只包含有效数据的叶子节点记录。而不同的行格式存储的行记录也不同，例如在早期版本中使用的redundant格式会被现在的compact格式使用更多的字节数来描述记录，例如描述记录的一些列信息，在使用compact格式时，可以改为直接从数据词典获取。因为redundant属于渐渐被抛弃的格式，本文的讨论中我们默认使用Compact格式。在文件rem/rem0rec.cc的头部注释描述了记录的物理结构。

每个记录都存在rec header，描述如下（参阅文件include/rem0rec.ic）

| bytes | Desc
--------------- |---------
| 变长列长度数组 | 如果列的最大长度为255字节，使用1byte；否则，0xxxxxxx (one byte, length=0..127), or 1exxxxxxxxxxxxxx (two bytes, length=128..16383, extern storage flag)
| SQL-NULL flag | 标示值为NULL的列的bitmap，每个位标示一个列，bitmap的长度取决于索引上可为NULL的列的个数(dict_index_t::n_nullable)，这两个数组的解析可以参阅函数`rec_init_offsets`
| 下面5个字节（REC_N_NEW_EXTRA_BYTES）描述记录的额外信息 | .... 
| REC_NEW_INFO_BITS (4 bits) | 目前只使用了两个bit，一个用于表示该记录是否被标记删除(`REC_INFO_DELETED_FLAG`)，另一个bit(REC_INFO_MIN_REC_FLAG)如果被设置，表示这个记录是当前level最左边的page的第一个用户记录
| REC_NEW_N_OWNED (4 bits) | 当该值为非0时，表示当前记录占用page directory里一个slot，并和前一个slot之间存在这么多个记录
| REC_NEW_HEAP_NO (13 bits) | 该记录的heap no
| REC_NEW_STATUS (3 bits) | 记录的类型，包括四种：`REC_STATUS_ORDINARY`(叶子节点记录)， `REC_STATUS_NODE_PTR`（非叶子节点记录），`REC_STATUS_INFIMUM`(infimum系统记录)以及`REC_STATUS_SUPREMUM`(supremum系统记录) 
| REC_NEXT (2bytes) | 指向按照键值排序的page内下一条记录数据起点，这里存储的是和当前记录的相对位置偏移量（函数`rec_set_next_offs_new`）

在记录头信息之后的数据视具体情况有所不同：

* 对于聚集索引记录，数据包含了事务id，回滚段指针；
* 对于二级索引记录，数据包含了二级索引键值以及聚集索引键值。如果二级索引键和聚集索引有重合，则只保留一份重合的，例如pk (col1, col2)，sec key(col2, col3)，在二级索引记录中就只包含(col2, col3, col1);
* 对于非叶子节点页的记录，聚集索引上包含了其子节点的最小记录键值及对应的page no；二级索引上有所不同，除了二级索引键值外，还包含了聚集索引键值，再加上page no三部分构成。

**Free space**

这里指的是一块完整的未被使用的空间，范围在页内最后一个用户记录和Page directory之间。通常如果空间足够时，直接从这里分配记录空间。当判定空闲空间不足时，会做一次Page内的重整理，以对碎片空间进行合并。

**Page directory**

为了加快页内的数据查找，会按照记录的顺序，每隔4~8个数量（PAGE_DIR_SLOT_MIN_N_OWNED ~ PAGE_DIR_SLOT_MAX_N_OWNED）的用户记录，就分配一个slot （每个slot占用2个字节，`PAGE_DIR_SLOT_SIZE`），存储记录的页内偏移量，可以理解为在页内构建的一个很小的索引(sparse index)来辅助二分查找。

Page Directory的slot分配是从Page末尾（倒数第八个字节开始）开始逆序分配的。在查询记录时。先根据page directory 确定记录所在的范围，然后在据此进行线性查询。

增加slot的函数参阅 `page_dir_add_slot`

页内记录二分查找的函数参阅 `page_cur_search_with_match_bytes`

**FIL Trailer**

在每个文件页的末尾保留了8个字节（FIL_PAGE_DATA_END or FIL_PAGE_END_LSN_OLD_CHKSUM），其中4个字节用于存储page checksum，这个值需要和page头部记录的checksum相匹配，否则认为page损坏(`buf_page_is_corrupted`)

###压缩索引页

InnoDB当前存在两种形式的压缩页，一种是Transparent Page Compression，还有一种是传统的压缩方式，下文分别进行阐述。

####Transparent Page Compression

这是MySQL5.7新加的一种数据压缩方式，其原理是利用内核Punch hole特性，对于一个16kb的数据页，在写文件之前，除了Page头之外，其他部分进行压缩，压缩后留白的地方使用punch hole进行 “打洞”，在磁盘上表现为不占用空间 （但会产生大量的磁盘碎片）。 这种方式相比传统的压缩方式具有更好的压缩比，实现逻辑也更加简单。

对于这种压缩方式引入了新的类型`FIL_PAGE_COMPRESSED`，在存储格式上略有不同，主要表现在从FIL_PAGE_FILE_FLUSH_LSN开始的8个字节被用作记录压缩信息：

| Macro | bytes | Desc
--------------- |---------| ------
| FIL_PAGE_VERSION | 1 | 版本，目前为1
| FIL_PAGE_ALGORITHM_V1 | 1 | 使用的压缩算法
| FIL_PAGE_ORIGINAL_TYPE_V1 | 2 | 压缩前的Page类型，解压后需要恢复回去
| FIL_PAGE_ORIGINAL_SIZE_V1 | 2 | 未压缩时去除FIL_PAGE_DATA后的数据长度
| FIL_PAGE_COMPRESS_SIZE_V1 | 2 | 压缩后的长度

打洞后的page其实际存储空间需要是磁盘的block size的整数倍。

这里我们不展开阐述，具体参阅我之前写的这篇文章：[MySQL · 社区动态 · InnoDB Page Compression](http://mysql.taobao.org/monthly/2015/08/01/)

####传统压缩存储格式

当你创建或修改表，指定`row_format=compressed key_block_size=1|2|4|8` 时，创建的ibd文件将以对应的block size进行划分。例如key_block_size设置为4时，对应block size为4kb。

压缩页的格式可以描述如下表所示：

| Macro | Desc
--------------- |---------
| FIL_PAGE_HEADER | 页面头数据，不做压缩
| Index Field Information | 索引的列信息，参阅函数`page_zip_fields_encode`及`page_zip_fields_decode`，在崩溃恢复时可以据此恢复出索引信息
| Compressed Data | 压缩数据，按照heap no排序进入压缩流，压缩数据不包含系统列(trx_id, roll_ptr)或外部存储页指针
| Modification Log(mlog) | 压缩页修改日志
| Free Space | 空闲空间
| External_Ptr (optional) | 存在外部存储页的列记录指针数组，只存在**聚集索引叶子节点**，每个数组元素占20个字节(`BTR_EXTERN_FIELD_REF_SIZE`)，参阅函数`page_zip_compress_clust_ext`
| Trx_id, Roll_Ptr(optional) | 只存在于**聚集索引叶子节点**，数组元素和其heap no一一对应
| Node_Ptr | 只存在于**索引非叶子节点**，存储节点指针数组，每个元素占用4字节(REC_NODE_PTR_SIZE)
| Dense Page Directory | 分两部分，第一部分是有效记录，记录其在解压页中的偏移位置，n_owned和delete标记信息，按照**键值顺序**；第二部分是空闲记录；每个slot占两个字节。

在内存中通常存在压缩页和解压页两份数据。当对数据进行修改时，通常先修改解压页，再将DML操作以一种特殊日志的格式记入压缩页的mlog中。以减少被修改过程中重压缩的次数。主要包含这几种操作：

* Insert: 向mlog中写入完整记录
* Update: 
   * Delete-insert update，将旧记录的dense slot标记为删除，再写入完整新记录
   * In-place update，直接写入新更新的记录
* Delete: 标记对应的dense slot为删除

页压缩参阅函数 `page_zip_compress`
页解压参阅函数 `page_zip_decompress`

### 系统数据页

这里我们将所有非独立的数据页统称为系统数据页，主要存储在ibdata中，如下图所示：

![3](http://mysql.taobao.org/monthly/pic/2016-02-01/3.png)

ibdata的三个page和普通的用户表空间一样，都是用于维护和管理文件页。其他Page我们下面一一进行介绍。

**FSP_IBUF_HEADER_PAGE_NO**

Ibdata的第4个page是Change Buffer的header page，类型为FIL_PAGE_TYPE_SYS，主要用于对ibuf btree的Page管理。

**FSP_IBUF_TREE_ROOT_PAGE_NO**

用于存储change buffer的根page，change buffer目前存储于Ibdata中，其本质上也是一颗btree，root页为固定page，也就是Ibdata的第5个page。

IBUF HEADER Page 和Root Page联合起来对ibuf的数据页进行管理。

首先Ibuf btree自己维护了一个空闲Page链表，链表头记录在根节点中，偏移量在PAGE_BTR_IBUF_FREE_LIST处，实际上利用的是普通索引根节点的PAGE_BTR_SEG_LEAF字段。Free List上的Page类型标示为`FIL_PAGE_IBUF_FREE_LIST`

每个Ibuf page重用了PAGE_BTR_SEG_LEAF字段，以维护IBUF FREE LIST的前后文件页节点（PAGE_BTR_IBUF_FREE_LIST_NODE）

由于root page中的segment字段已经被重用，因此额外的开辟了一个Page，也就是Ibdata的第4个page来进行段管理。在其中记录了ibuf btree的segment header，指向属于ibuf btree的inode entry。

关于ibuf btree的构建参阅函数 `btr_create`

**FSP_TRX_SYS_PAGE_NO/FSP_FIRST_RSEG_PAGE_NO**

ibdata的第6个page，记录了InnoDB重要的事务系统信息，主要包括：

| Macro | bytes | Desc
--------------- |---------| ------
| TRX_SYS | 38 | 每个数据页都会保留的文件头字段
| TRX_SYS_TRX_ID_STORE | 8 | 持久化的最大事务ID，这个值不是实时写入的，而是256次递增写一次
| TRX_SYS_FSEG_HEADER | 10 | 指向用来管理事务系统的segment所在的位置
| TRX_SYS_RSEGS | 128 * 8 | 用于存储128个回滚段位置，包括space id及page no。每个回滚段包含一个文件segment（`trx_rseg_header_create`）
| …… | 以下是Page内UNIV_PAGE_SIZE - 1000的偏移位置 |
| TRX_SYS_MYSQL_LOG_MAGIC_N_FLD | 4 | Magic Num ，值为873422344
| TRX_SYS_MYSQL_LOG_OFFSET_HIGH | 4 | 事务提交时会将其binlog位点更新到该page中，这里记录了在binlog文件中偏移量的高位的4字节
| TRX_SYS_MYSQL_LOG_OFFSET_LOW | 4 | 同上，记录偏移量的低4位字节
| TRX_SYS_MYSQL_LOG_NAME | 4 | 记录所在的binlog文件名
| …… | 以下是Page内UNIV_PAGE_SIZE - 200 的偏移位置 |
| TRX_SYS_DOUBLEWRITE_FSEG | 10 | 包含double write buffer的fseg header
| TRX_SYS_DOUBLEWRITE_MAGIC | 4 | Magic Num
| TRX_SYS_DOUBLEWRITE_BLOCK1 | 4 | double write buffer的第一个block(占用一个Extent)在ibdata中的开始位置，连续64个page
| TRX_SYS_DOUBLEWRITE_BLOCK2 | 4 | 第二个dblwr block的起始位置
| TRX_SYS_DOUBLEWRITE_REPEAT | 12 | 重复记录上述三个字段，即MAGIC NUM, block1, block2，防止发生部分写时可以恢复
| TRX_SYS_DOUBLEWRITE_SPACE_ID_STORED | 4 | 用于兼容老版本，当该字段的值不为TRX_SYS_DOUBLEWRITE_SPACE_ID_STORED_N时，需要重置dblwr中的数据

在5.7版本中，回滚段既可以在ibdata中，也可以在独立undo表空间，或者ibtmp临时表空间中，一个可能的分布如下图所示（摘自我之前的[这篇文章](http://mysql.taobao.org/monthly/2015/04/01/)）。

![4](http://mysql.taobao.org/monthly/pic/2016-02-01/4.png)

由于是在系统刚启动时初始化事务系统，因此第0号回滚段头页总是在ibdata的第7个page中。

事务系统创建参阅函数 `trx_sysf_create`

InnoDB最多可以创建128个回滚段，每个回滚段需要单独的Page来维护其拥有的undo slot，Page类型为FIL_PAGE_TYPE_SYS。描述如下：

| Macro | bytes | Desc
--------------- |---------| ------
| TRX_RSEG | 38 | 保留的Page头
| TRX_RSEG_MAX_SIZE | 4 | 回滚段允许使用的最大Page数，当前值为ULINT_MAX
| TRX_RSEG_HISTORY_SIZE | 4 | 在history list上的undo page数，这些page需要由purge线程来进行清理和回收
| TRX_RSEG_HISTORY | FLST_BASE_NODE_SIZE(16) | history list的base node
| TRX_RSEG_FSEG_HEADER | (FSEG_HEADER_SIZE)10 | 指向当前管理当前回滚段的inode entry
| TRX_RSEG_UNDO_SLOTS | 1024 * 4 | undo slot数组，共1024个slot，值为FIL_NULL表示未被占用，否则记录占用该slot的第一个undo page

回滚段头页的创建参阅函数 `trx_rseg_header_create`

实际存储undo记录的Page类型为FIL_PAGE_UNDO_LOG，undo header结构如下

| Macro | bytes | Desc
--------------- |---------| ------
| TRX_UNDO_PAGE_HDR | 38 | Page 头
| TRX_UNDO_PAGE_TYPE | 2 | 记录Undo类型，是TRX_UNDO_INSERT还是TRX_UNDO_UPDATE
| TRX_UNDO_PAGE_START | 2 | 事务所写入的最近的一个undo log在page中的偏移位置
| TRX_UNDO_PAGE_FREE | 2 | 指向当前undo page中的可用的空闲空间起始偏移量
| TRX_UNDO_PAGE_NODE | 12 | 链表节点，提交后的事务，其拥有的undo页会加到history list上

undo页内结构及其与回滚段头页的关系参阅下图：

![5](http://mysql.taobao.org/monthly/pic/2016-02-01/5.png)

关于具体的Undo log如何存储，本文不展开描述，可阅读我之前的这篇文章：[MySQL · 引擎特性 · InnoDB undo log 漫游](http://mysql.taobao.org/monthly/2015/04/01/)

**FSP_DICT_HDR_PAGE_NO**

ibdata的第8个page，用来存储数据词典表的信息 （只有拿到数据词典表，才能根据其中存储的表信息，进一步找到其对应的表空间，以及表的聚集索引所在的page no）

Dict_Hdr Page的结构如下表所示：

| Macro | bytes | Desc
--------------- |---------| ------
| DICT_HDR | 38 | Page头
| DICT_HDR_ROW_ID | 8 | 最近被赋值的row id，递增，用于给未定义主键的表，作为其隐藏的主键键值来构建btree
| DICT_HDR_TABLE_ID | 8 | 当前系统分配的最大事务ID，每创建一个新表，都赋予一个唯一的table id，然后递增
| DICT_HDR_INDEX_ID | 8 | 用于分配索引ID
| DICT_HDR_MAX_SPACE_ID | 4 | 用于分配space id
| DICT_HDR_MIX_ID_LOW | 4 |
| DICT_HDR_TABLES | 4 | SYS_TABLES系统表的聚集索引root page
| DICT_HDR_TABLE_IDS | 4 | SYS_TABLE_IDS索引的root page
| DICT_HDR_COLUMNS | 4 | SYS_COLUMNS系统表的聚集索引root page
| DICT_HDR_INDEXES | 4 | SYS_INDEXES系统表的聚集索引root page
| DICT_HDR_FIELDS | 4 | SYS_FIELDS系统表的聚集索引root page

dict_hdr页的创建参阅函数 `dict_hdr_create`

**double write buffer**

InnoDB使用double write buffer来防止数据页的部分写问题，在写一个数据页之前，总是先写double write buffer，再写数据文件。当崩溃恢复时，如果数据文件中page损坏，会尝试从dblwr中恢复。

double write buffer存储在ibdata中，你可以从事务系统页(ibdata的第6个page)获取dblwr所在的位置。总共128个page，划分为两个block。由于dblwr在安装实例时已经初始化好了，这两个block在Ibdata中具有固定的位置，Page64 ~127 划属第一个block，Page 128 ~191划属第二个block。

在这128个page中，前120个page用于batch flush时的脏页回写，另外8个page用于SINGLE PAGE FLUSH时的脏页回写。

###外部存储页

对于大字段，在满足一定条件时InnoDB使用外部页进行存储。外部存储页有三种类型：

1. FIL_PAGE_TYPE_BLOB：表示非压缩的外部存储页，结构如下图所示：

![6](http://mysql.taobao.org/monthly/pic/2016-02-01/6.png)

2. FIL_PAGE_TYPE_ZBLOB：压缩的外部存储页，如果存在多个blob page，则表示第一个
   FIL_PAGE_TYPE_ZBLOB2：如果存在多个压缩的blob page，则表示blob链随后的page；
   结构如下图所示：

![7](http://mysql.taobao.org/monthly/pic/2016-02-01/7.png) 

而在记录内只存储了20个字节的指针以指向外部存储页，指针描述如下：

| Macro | bytes | Desc
--------------- |---------| ------
| BTR_EXTERN_SPACE_ID | 4 | 外部存储页所在的space id
| BTR_EXTERN_PAGE_NO | 4 | 第一个外部页的Page no
| BTR_EXTERN_OFFSET | 4 | 对于压缩页，为12，该偏移量存储了指向下一个外部页的的page no；对于非压缩页，值为38，指向blob header，如上图所示 
   
外部页的写入参阅函数 `btr_store_big_rec_extern_fields`

### MySQL5.7新数据页：加密页及R-TREE页

MySQL 5.7版本引入了新的数据页以支持表空间加密及对空间数据类型建立R-TREE索引。本文对这种数据页不做深入讨论，仅仅简单描述下，后面我们会单独开两篇文章分别进行介绍。

**数据加密页**

从MySQL5.7.11开始InnoDB支持对单表进行加密，因此引入了新的Page类型来支持这一特性，主要加了三种Page类型：

* FIL_PAGE_ENCRYPTED：加密的普通数据页
* FIL_PAGE_COMPRESSED_AND_ENCRYPTED:数据页为压缩页(transparent page compression) 并且被加密（先压缩，再加密）
* FIL_PAGE_ENCRYPTED_RTREE：GIS索引R-TREE的数据页并被加密

对于加密页，除了数据部分被替换成加密数据外，其他部分和大多数表都是一样的结构。

加解密的逻辑和Transparent Compression类似，在写入文件前加密(`os_file_encrypt_page --> Encryption::encrypt`)，在读出文件时解密数据(`os_file_io_complete --> Encryption::decrypt`)

秘钥信息存储在ibd文件的第一个page中（`fsp_header_init --> fsp_header_fill_encryption_info`），当执行SQL `ALTER INSTANCE ROTATE INNODB MASTER KEY`时，会更新每个ibd存储的秘钥信息(`fsp_header_rotate_encryption`)

默认安装时，一个新的插件keyring_file被安装并且默认Active，在安装目录下，会产生一个新的文件来存储秘钥，位置在$MYSQL_INSTALL_DIR/keyring/keyring，你可以通过参数[keyring_file_data](http://dev.mysql.com/doc/refman/5.7/en/server-system-variables.html#sysvar_keyring_file_data)来指定秘钥的存放位置和文件命名。 当你安装多实例时，需要为不同的实例指定keyring文件。

开启表加密的语法很简单，在CREATE TABLE或ALTER TABLE时指定选项ENCRYPTION=‘Y’来开启，或者ENCRYPTION=‘N’来关闭加密。

关于InnoDB表空间加密特性，参阅该[commit](https://github.com/mysql/mysql-server/commit/9340eb1146fedc538cc54e96a45f95a58b345fbf)及[官方文档](http://dev.mysql.com/doc/refman/5.7/en/innodb-tablespace-encryption.html)

**R-TREE索引页**

在MySQL 5.7中引入了新的索引类型R-TREE来描述空间数据类型的多维数据结构，这类索引的数据页类型为`FIL_PAGE_RTREE`。

R-TREE的相关设计参阅官方[WL#6968](http://dev.mysql.com/worklog/task/?id=6968)， [WL#6609](http://dev.mysql.com/worklog/task/?id=6609), [WL#6745](http://dev.mysql.com/worklog/task/?id=6745)

###临时表空间ibtmp

MySQL5.7引入了临时表专用的表空间，默认命名为ibtmp1，创建的非压缩临时表都存储在该表空间中。系统重启后，ibtmp1会被重新初始化到默认12MB。你可以通过设置参数[innodb_temp_data_file_path](http://dev.mysql.com/doc/refman/5.7/en/innodb-parameters.html#sysvar_innodb_temp_data_file_path)来修改ibtmp1的默认初始大小，以及是否允许autoExtent。默认值为 ”ibtmp1:12M:autoExtent“

除了用户定义的非压缩临时表外，第1~32个临时表专用的回滚段也存放在该文件中（0号回滚段总是存放在ibdata中）(`trx_sys_create_noredo_rsegs`)，

###日志文件ib_logfile

关于日志文件的格式，网上已经有很多的讨论，在之前的[系列文章](http://mysql.taobao.org/monthly/2015/05/01/)中我也有专门介绍过，本小节主要介绍下MySQL5.7新的修改。

首先是checksum算法的改变，当前版本的MySQL5.7可以通过参数innodb_log_checksums来开启或关闭redo checksum，但目前唯一支持的checksum算法是CRC32。而在之前老版本中只支持效率较低的InnoDB本身的checksum算法。

第二个改变是为Redo log引入了版本信息([WL#8845](http://dev.mysql.com/worklog/task/?id=8845))，存储在ib_logfile的头部，从文件头开始，描述如下

| Macro | bytes | Desc
--------------- |---------| ------
| LOG_HEADER_FORMAT | 4 | 当前值为1(LOG_HEADER_FORMAT_CURRENT)，在老版本中这里的值总是为0
| LOG_HEADER_PAD1 | 4 | 新版本未使用
| LOG_HEADER_START_LSN | 8 | 当前iblogfile的开始LSN
| LOG_HEADER_CREATOR | 32 | 记录版本信息，和MySQL版本相关，例如在5.7.11中，这里存储的是"MySQL 5.7.11"(LOG_HEADER_CREATOR_CURRENT)

每次切换到下一个iblogfile时，都会更新该文件头信息(`log_group_file_header_flush`)

新的版本支持兼容老版本（`recv_find_max_checkpoint_0`），但升级到新版本后，就无法在异常状态下in-place降级到旧版本了（除非做一次clean的shutdown，并清理掉iblogfile）。

具体实现参阅该[commit](https://github.com/mysql/mysql-server/commit/af0acedd885eb7103e319f79d25fda7386ef1506)

## IO子系统

本小节我们介绍下磁盘文件与内存数据的中枢，即IO子系统。InnoDB对page的磁盘操作分为读操作和写操作。

对于读操作，在将数据读入磁盘前，总是为其先预先分配好一个block，然后再去磁盘读取一个新的page，在使用这个page之前，还需要检查是否有change buffer项，并根据change buffer进行数据变更。读操作分为两种场景：普通的读page及预读操作，前者为同步读，后者为异步读
 
数据写操作也分为两种，一种是batch write，一种是single page write。写page默认受double write buffer保护，因此对double write buffer的写磁盘为同步写，而对数据文件的写入为异步写。
 
同步读写操作通常由用户线程来完成，而异步读写操作则需要后台线程的协同。
 
举个简单的例子，假设我们向磁盘批量写数据，首先先写到double write buffer，当dblwr满了之后，一次性将dblwr中的数据同步刷到Ibdata，在确保sync到dblwr后，再将这些page分别异步写到各自的文件中。注意这时候dblwr依旧未被清空，新的写Page请求会进入等待。当异步写page完成后，io helper线程会调用buf_flush_write_complete，将写入的Page从flush list上移除。当dblwr中的page完全写完后，在函数buf_dblwr_update里将dblwr清空。这时候才允许新的写请求进dblwr。

同样的，对于异步写操作，也需要IO Helper线程来检查page是否完好、merge change buffer等一系列操作。

除了数据页的写入，还包括日志异步写入线程、及ibuf后台线程。
 
###IO后台线程

InnoDB的IO后台线程主要包括如下几类：

* IO READ 线程： 后台读线程，线程数目通过参数innodb_read_io_threads配置，主要处理INNODB 数据文件异步读请求，任务队列为`AIO::s_reads`，任务队列包含slot数为线程数 * 256(linux 平台)，也就是说，每个read线程最多可以pend 256个任务；
* IO WRITE 线程： 后台写线程数，线程数目通过参数innodb_write_io_threads配置。主要处理INNODB 数据文件异步写请求，任务队列为`AIO::s_writes`，任务队列包含slot数为线程数 * 256(linux 平台)，也就是说，每个read线程最多可以pend 256个任务；
* LOG 线程：写日志线程。只有在写checkpoint信息时才会发出一次异步写请求。任务队列为`AIO::s_log`，共1个segment，包含256个slot；
* IBUF 线程：负责读入change buffer页的后台线程，任务队列为`AIO::s_ibuf`，共1个segment，包含256个slot
 
所有的同步写操作都是由用户线程或其他后台线程执行。上述IO线程只负责异步操作。
 
###发起IO请求
 
入口函数：`os_aio_func`
 
首先对于同步读写请求（OS_AIO_SYNC），发起请求的线程直接调用os_file_read_func 或者os_file_write_func 去读写文件 ，然后返回。

对于异步请求，用户线程从对应操作类型的任务队列（AIO::select_slot_array）中选取一个slot，将需要读写的信息存储于其中（AIO::reserve_slot）:

* 首先在任务队列数组中选择一个segment；这里根据偏移量来算segment，因此可以尽可能的将相邻的读写请求放到一起，这有利于在IO层的合并操作

```
local_seg = (offset >> (UNIV_PAGE_SIZE_SHIFT + 6)) % m_n_segments;
```
 
* 从该segment范围内选择一个空闲的slot，如果没有则等待；
* 将对应的文件读写请求信息赋值到slot中，例如写入的目标文件，偏移量，数据等；
* 如果这是一次IO写入操作，且使用native aio时，如果表开启了transparent compression，则对要写入的数据页先进行压缩并punch hole；如果设置了表空间加密，再对数据页进行加密；

对于Native AIO （使用linux自带的LIBAIO库），调用函数AIO::linux_dispatch，将IO请求分发给kernel层。
 
如果没有开启Native AIO，且没有设置wakeup later 标记，则会去唤醒io线程（AIO::wake_simulated_handler_thread），这是早期libaio还不成熟时，InnoDB在内部模拟aio实现的逻辑。
 
Tips：编译Native AIO需要安装libaio-dev包，并打开选项srv_use_native_aio

###处理异步AIO请求
 
IO线程入口函数为``io_handler_thread --> fil_aio_wait``

首先调用`os_aio_handler`来获取请求： 

* 对于Native AIO，调用函数os_aio_linux_handle 获取读写请求。 IO线程会反复以500ms（`OS_AIO_REAP_TIMEOUT`）的超时时间通过io_getevents确认是否有任务已经完成了（`LinuxAIOHandler::collect()`），如果有读写任务完成，找到已完成任务的slot后，释放对应的槽位。
* 对于simulated aio，调用函数`os_aio_simulated_handler` 处理读写请求，这里相比NATIVE AIO要复杂很多
    * 如果这是异步读队列，并且`os_aio_recommend_sleep_for_read_threads`被设置，则暂时不处理，而是等待一会，让其他线程有机会将更过的IO请求发送过来。目前linear readhaed 会使用到该功能。这样可以得到更好的IO合并效果。(`SimulatedAIOHandler::check_pending`)
    * 已经完成的slot需要及时被处理(`SimulatedAIOHandler::check_completed`，可能由上次的io合并操作完成)
    * 如果有超过2秒未被调度的请求(`SimulatedAIOHandler::select_oldest`)，则优先选择最老的slot，防止饿死，否则，找一个文件读写偏移量最小的位置的slot(SimulatedAIOHandler::select())
    * 没有任何请求时进入等待状态
    * 当找到一个未完成的slot时，会尝试merge相邻的IO请求（`SimulatedAIOHandler::merge()`），并将对应的slot加入到SimulatedAIOHandler::m_slots数组中，最多不超过64个slot
    * 然而在5.7版本里，合并操作已经被禁止了，全部改成了一个个slot进行读写，升级到5.7的用户一定要注意这个改变，或者改为使用更好的Native AIO方式
    * 完成io后，释放slot; 并选择第一个处理完的slot作为随后优先完成的请求。


从上一步获得完成IO的slot后，调用函数`fil_node_complete_io`， 递减`node->n_pending`。 对于文件写操作，需要加入到fil_system->unflushed_spaces链表上，表示这个文件修改过了，后续需要被sync到磁盘。
 
如果设置为O_DIRECT_NO_FSYNC的文件IO模式，则数据文件无需加入到fil_system_t::unflushed_spaces链表上。通常我们即时使用O_DIRECT的方式操作文件，也需要做一次sync，来保证文件元数据的持久化，但在某些文件系统下则没有这个必要，通常只要文件的大小这些关键元数据没发生变化，可以省略一次fsync。

最后在IO完成后，调用`buf_page_io_complete`，做page corruption检查、change buffer merge等操作；对于写操作，需要从flush list上移除block并更新double write buffer；对于LRU FLUSH产生的写操作，还会将其对应的block释放到free list上；

对于日志文件操作，调用`log_io_complete`执行一次fil_flush，并更新内存内的checkpoint信息（`log_complete_checkpoint`）
 
###IO 并发控制
 
由于文件底层使用pwrite/pread来进行文件I/O，因此用户线程对文件普通的并发I/O操作无需加锁。但在windows平台下，则需要加锁进行读写。
 
对相同文件的IO操作通过大量的counter/flag来进行并发控制。
 
当文件处于扩展阶段时（`fil_space_extend`），将`fil_node_t::being_extended`设置为true，避免产生并发Extent，或其他关闭文件或者rename操作等
 
当正在删除一个表时，会检查是否有pending的操作（fil_check_pending_operations）

* 将fil_space_t::stop_new_ops设置为true；
* 检查是否有Pending的change buffer merge (fil_space_t::n_pending_ops)；有则等待
* 检查是否有pending的IO（fil_node_t::n_pending） 或者pending的文件flush操作（`fil_node_t::n_pending_flushes`）；有则等待
 
当truncate一张表时，和drop table类似，也会调用函数`fil_check_pending_operations`，检查表上是否有pending的操作，并将`fil_space_t::is_being_truncated`设置为true
 
当rename一张表时（`fil_rename_tablespace`），将文件的stop_ios标记设置为true，阻止其他线程所有的I/O操作
 
当进行文件读写操作时，如果是异步读操作，发现stop_new_ops或者被设置了但is_being_truncated未被设置，会返回报错；但依然允许同步读或异步写操作(`fil_io`)

当进行文件flush操作时，如果发现`fil_space_t::stop_new_ops`或者`fil_space_t::is_being_truncated`被设置了，则忽略该文件的flush操作 （`fil_flush_file_spaces`）

###文件预读
 
文件预读是一项在SSD普及前普通磁盘上比较常见的技术，通过预读的方式进行连续IO而非带价高昂的随机IO。InnoDB有两种预读方式：随机预读及线性预读； Facebook另外还实现了一种逻辑预读的方式

**随机预读**

入口函数：`buf_read_ahead_random`
 
以64个Page为单位(这也是一个Extent的大小)，当前读入的page no所在的64个pagno 区域[ (page_no/64)*64, (page_no/64) *64 + 64]，如果最近被访问的Page数超过BUF_READ_AHEAD_RANDOM_THRESHOLD（通常值为13），则将其他Page也读进内存。这里采取异步读。

随机预读受参数innodb_random_read_ahead控制

**线性预读**

入口函数：`buf_read_ahead_linear`
 
所谓线性预读，就是在读入一个新的page时，和随机预读类似的64个连续page范围内，默认从低到高Page no，如果最近连续被访问的page数超过innodb_read_ahead_threshold，则将该Extent之后的其他page也读取进来。

**逻辑预读**
 
由于表可能存在碎片空间，因此很可能对于诸如全表扫描这样的场景，连续读取的page并不是物理连续的，线性预读不能解决这样的问题，另外一次读取一个Extent对于需要全表扫描的负载并不足够。因此facebook引入了逻辑预读。
 
其大致思路为，扫描聚集索引，搜集叶子节点号，然后根据叶子节点的page no (可以从非叶子节点获取)顺序异步读入一定量的page。
 
由于Innodb aio一次只支持提交一个page读请求，虽然Kernel层本身会做读请求合并，但那显然效率不够高。他们对此做了修改，使INNODB可以支持一次提交（io_submit）多个aio请求。

入口函数：`row_search_for_mysql --> row_read_ahead_logical`

具体参阅[这篇博文](http://planet.mysql.com/entry/?id=516236)
 
或者webscalesql上的几个commit：

```
git show 2d61329446a08f85c89a4119317ae85baacf2bbb   // 合并多个AIO请求，对所有的预读逻辑（上述三种）采用这种方式
git show 9f52bfd2222403f841fe5fcbedd1333f78a70a4b     //  逻辑预读的主要代码逻辑
git show 64b68e07430b50f6bff5ed67374b336623db24b6   // 防止事务在多个表上读取操作时预读带来的影响
```

###日志填充写入
 
由于现代磁盘通常的block size都是大于512字节的，例如一般是4096字节，为了避免 “read-on-write” 问题，在5.7版本里添加了一个参数innodb_log_write_ahead_size，你可以通过配置该参数，在写入redo log时，将写入区域配置到block size对齐的字节数。
 
在代码里的实现，就是在写入redo log 文件之前，为尾部字节填充0，（参考函数log_write_up_to）
 
Tips：所谓READ-ON-WRITE问题，就是当修改的字节不足一个block时，需要将整个block读进内存，修改对应的位置，然后再写进去；如果我们以block为单位来写入的话，直接完整覆盖写入即可。

##buffer pool内存管理

InnoDB buffer pool从5.6到5.7版本发生了很大的变化。首先是分配方式上不同，其次实现了更好的刷脏效率。对buffer pool上的各个链表的管理也更加高效。

###buffer pool初始化

在5.7之前的版本中，一个buffer pool instance拥有一个chunk，每个chunk的大小为buffer pool size / instance个数。

而到了5.7版本中，每个instance可能划分成多个chunk，每个chunk的大小是可定义的，默认为127MB。因此一个buffer pool instance可能包含多个chunk内存块。这么做的目的是为了实现在线调整buffer pool大小([WL#6117](http://dev.mysql.com/worklog/task/?id=6117))，buffer pool增加或减少必须以chunk为基本单位进行。

在5.7里有个问题值得关注，即buffer pool size会根据instances * chunk size向上对齐，举个简单的例子，假设你配置了64个instance, chunk size为默认128MB，就需要以8GB进行对齐，这意味着如果你配置了9GB的buffer pool，实际使用的会是16GB。所以**尽量不要配置太多的buffer pool instance**

###buffer pool链表及管理对象

出于不同的目的，每个buffer pool instance上都维持了多个链表，可以根据space id及page no找到对应的instance(`buf_pool_get`)。

一些关键的结构对象及描述如下表所示：

| name | desc 
-------|-----
| buf_pool_t::page_hash | page_hash用于存储已经或正在读入内存的page。根据<space_id, page_no>快速查找。当不在page hash时，才会去尝试从文件读取
| buf_pool_t::LRU | LRU上维持了所有从磁盘读入的数据页，该LRU上又在链表尾部开始大约3/8处将链表划分为两部分，新读入的page被加入到这个位置；当我们设置了innodb_old_blocks_time，若两次访问page的时间超过该阀值，则将其挪动到LRU头部；这就避免了类似一次性的全表扫描操作导致buffer pool污染
| buf_pool_t::free | 存储了当前空闲可分配的block
| buf_pool_t::flush_list | 存储了被修改过的page，根据oldest_modification（即载入内存后第一次修改该page时的Redo LSN）排序
| buf_pool_t::flush_rbt | 在崩溃恢复阶段在flush list上建立的红黑数，用于将apply redo后的page快速的插入到flush list上，以保证其有序
| buf_pool_t::unzip_LRU | 压缩表上解压后的page被存储到unzip_LRU。 buf_block_t::frame存储解压后的数据，buf_block_t::page->zip.data指向原始压缩数据。
| buf_pool_t::zip_free[BUF_BUDDY_SIZES_MAX] | 用于管理压缩页产生的空闲碎片page。压缩页占用的内存采用buddy allocator算法进行分配。

###buffer pool并发控制

除了不同的用户线程会并发操作buffer pool外，还有后台线程也会对buffer pool进行操作。InnoDB通过读写锁，buf fix计数，io fix标记来进行并发控制。

**读写并发控制**

通常当我们读取到一个page时，会对其加block S锁，并递增buf_page_t::buf_fix_count，直到mtr commit时才会恢复。而如果读page的目的是为了进行修改，则会加X锁

当一个page准备flush到磁盘时(`buf_flush_page`)，如果当前Page正在被访问，其buf_fix_count不为0时，就忽略flush该page，以减少获取block上SX Lock的昂贵代价。

**并发读控制**

当多个线程请求相同的page时，如果page不在内存，是否可能引发对同一个page的文件IO ？答案是不会。

从函数`buf_page_init_for_read`我们可以看到，在准备读入一个page前，会做如下工作：

* 分配一个空闲block
* buf_pool_mutex_enter
* 持有page_hash x lock
* 检查page_hash中是否已被读入，如果是，表示另外一个线程已经完成了io，则忽略本次io请求，退出
* 持有block->mutex，对block进行初始化，并加入到page hash中
* 设置IO FIX为BUF_IO_READ
* 释放hash lock
* 将block加入到LRU上
* 持有block s lock
* 完成IO后，释放s lock

当另外一个线程也想请求相同page时，首先如果看到page hash中已经有对应的block了，说明page已经或正在被读入buffer pool，如果io_fix为BUF_IO_READ，说明正在进行IO，就通过加X锁的方式做一次sync（`buf_wait_for_read`），确保IO完成。

请求Page通常还需要加S或X锁，而IO期间也是持有block x锁的，如果成功获取了锁，说明IO肯定完成了。


###Page驱逐及刷脏

当buffer pool中的free list不足时，为了获取一个空闲block，通常会触发page驱逐操作(`buf_LRU_free_from_unzip_LRU_list`)

首先由于压缩页在内存中可能存在两份拷贝：压缩页和解压页；InnoDB根据最近的IO情况和数据解压技术来判定实例是处于IO-BOUND还是CPU-BOUND（`buf_LRU_evict_from_unzip_LRU`）。如果是IO-BOUND的话，就尝试从unzip_lru上释放一个block出来(`buf_LRU_free_from_unzip_LRU_list`)，而压缩页依旧保存在内存中。

其次再考虑从buf_pool_t::LRU链表上释放block，如果有可替换的page(`buf_flush_ready_for_replace`)时，则将其释放掉，并加入到free list上；对于压缩表，压缩页和解压页在这里都会被同时驱逐。

当无法从LRU上获得一个可替换的Page时，说明当前Buffer pool可能存在大量脏页，这时候会触发single page flush(`buf_flush_single_page_from_LRU`)，即用户线程主动去刷一个脏页并替换掉。这是个慢操作，尤其是如果并发很高的时候，可能观察到系统的性能急剧下降。在RDS MySQL中，我们开启了一个后台线程， 能够自动根据当前Free List的长度来主动做flush，避免用户线程陷入其中。

除了single page flush外，在MySQL 5.7版本里还引入了多个page cleaner线程，根据一定的启发式算法，可以定期且高效的的做page flush操作。

本文对此不展开讨论，感兴趣的可以阅读我之前的月报：
[MySQL · 性能优化· 5.7.6 InnoDB page flush 优化](http://mysql.taobao.org/index.php?title=MySQL%E5%86%85%E6%A0%B8%E6%9C%88%E6%8A%A5_2015.03#MySQL_.C2.B7_.E6.80.A7.E8.83.BD.E4.BC.98.E5.8C.96.C2.B7_5.7.6_InnoDB_page_flush_.E4.BC.98.E5.8C.96)
[MySQL · 性能优化· InnoDB buffer pool flush策略漫谈](http://mysql.taobao.org/index.php?title=MySQL%E5%86%85%E6%A0%B8%E6%9C%88%E6%8A%A5_2015.02#MySQL_.C2.B7_.E6.80.A7.E8.83.BD.E4.BC.98.E5.8C.96.C2.B7_InnoDB_buffer_pool_flush.E7.AD.96.E7.95.A5.E6.BC.AB.E8.B0.88)
