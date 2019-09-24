## HBase实现原理

### HBase功能组件
HBase实现主要功能组件：

1. 库函数：链接到每个客户端
2. 一个Master主服务器：负责管理和维护HBase表的分区信息，维护Region服务器列表，分配Region，负载均衡
3. 多个Region服务器：负责存储和维护分配给自己的Region，处理来自客户端的读写请求

客户端不直接从Master主服务器读取数据，而是在获得Region的存储位置信息后，从Region服务器上读取数据

客户端通过ZooKeeper获得Region信息位置，大多数客户端不和Master通信，这种设计方式使Master负载很小

### 表和Region
开始只有一个Region，后来不断分裂

Region拆分操作非常快，接近瞬间，因为拆分之后的Region读取的仍然是原存储文件，直到“合并”过程把存储文件异步地写到独立的文件之后，才会读取新文件

![Region分裂]()

每个Region默认大小是100MB到200MB（2006年以前的硬件配置）

- 每个Region的最佳大小取决于单台服务器的有效处理能力
- 目前每个Region最佳大小建议1GB-2GB（2013年以后的硬件配置）

同一个Region不会被分拆到多个Region服务器

每个Region服务器存储10-1000个Region

![Region分布]()

### Region定位

![HBase三层结构]()

层次 | 名称 | 作用
--- | --- | ---
第一层 | ZooKeeper文件 | 记录-ROOT-表的位置信息
第二层 | -ROOT-表 | 根数据表。记录所有元数据的具体位置，只有唯一一个Region
第三层 | .META.表 | 元数据表。存储了Region和Region服务器的映射关系。当HBase表很大时， .META.表也会被分裂成多个Region

三层结构可以保存Region数目计算：

- 为了加快访问速度，.META.表的全部Region都会被保存在内存中
- 假设.META.表的每行（一个映射条目）在内存中大约占用1KB，并且每个Region限制为128MB
- Region数目的计算：-ROOT-表能够寻址的.META.表的Region个数）×（每个.META.表的 Region可以寻址的用户数据表的Region个数）
- 一个-ROOT-表最多只能有一个Region（最多只能有128MB），按照每行（一个映射条目）占用1KB内存计算，128MB空间可以容纳128MB/1KB= $2^{17}$ = 131072 行，**10万+行**（一个-ROOT-表可以寻址$2^{17}$个.META.表的Region）
- 每个.META.表的 Region可以寻址的用户数据表的Region个数是128MB/1KB=$2^{17}$
- 三层结构可以保存的Region数目是(128MB/1KB) × (128MB/1KB) = $2^{34}$ = 17179869184  **1百亿+**个Region