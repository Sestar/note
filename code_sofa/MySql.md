# sql

<br />

## 查询表的所有字段，类型，备注，key</span>
```sql
SELECT column_name columnName, data_type dataType, column_comment columnComment, column_key columnKey, extra
        FROM information_schema.columns
        WHERE table_name = 'order_item' AND table_schema = (SELECT database()) ORDER BY ordinal_position
```
</br>

## 查询表的备注
```sql
SHOW FULL COLUMNS FROM table_name;
```
</br>

# 查询表的DDL(字段类型, 备注)

<img width="30%" src="/note/_v_images/code_sofa/sql/DDL.png" alt="DDL" />

# 统计SQL

```text
select 字段1, 字段2, count(*) as 总数, 
sum(case when 字段3=1 then 1 else 0 end) as 一总数,
sum(case when 字段3=0 then 1 else 0 end) as 零总数
from 表名
group by 字段1,字段2
```