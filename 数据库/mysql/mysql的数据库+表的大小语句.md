查询数据库的  库和表的大小





-- 查询数据库的占用数据大小---  所有的数据库  

select
round(sum(data_length + index_length) / 1024 / 1024 / 1024, 2) as `total size (gb)`
from
information_schema.tables
				where
table_schema = DATABASE();



-- 查询数据库的占用数据大小---  指定数据库
select
round(sum(data_length + index_length) / 1024 / 1024 / 1024, 2) as `total size (gb)`
from
information_schema.tables
				where
table_schema = "jhmk_waring";









-- 数据库下的表大小 排序

select
table_name as `table`,
round((data_length + index_length) / 1024 / 1024/ 1024, 2) as `size (gb)`
from
information_schema.tables
where
table_schema = "jhmk_waring"
order by
(data_length + index_length) desc;