### MongoDB的语法

|                           Mognodb                            |        mysql        |
| :----------------------------------------------------------: | :-----------------: |
|                            DB 库                             |   database 数据库   |
|                       Collection  集合                       |      Table  表      |
| Document    单个文档最大不能超过16MB，否则就应该考虑使用引用（DBRef）了，在主表里存储一个id值，指向另一个表中的id值 | row/record 行、记录 |
|                          file 字段                           |        colum        |
|                          Index 索引                          |     index 索引      |
|                  连表 embedding & Linkding                   |        Join         |
|                          Shard 分片                          |    parttion 分区    |
|                     Sharding key  分片键                     |    parttion key     |

