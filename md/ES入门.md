- 文档Document
  - 用户存储在es中的数据文档
- 索引Index
  - 由具有相同字段的文档列表组成。类似于mysql中的表

- 节点Node
  - 一个ES的运行实例,是集群的构成单元
- 集群Cluster
  - 由一个或多个节点组成，对外提供服务

### Document

Json Object,由字段(Field)组成,常见数据类型如下:

- 字符型: text, keyword
- 数值型: long, integer, short, byte, double, float,half_float,scaed_float
- 布尔:boolean
- 日期:date
- 二进制:binary
- 范围类型:interger_range,float_range,long_range,double_range,date_range