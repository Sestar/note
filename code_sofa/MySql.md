# sql

* <span style="color:orange">查询表的所有字段，类型，备注，key</span>
```sql
SELECT column_name columnName, data_type dataType, column_comment columnComment, column_key columnKey, extra
        FROM information_schema.columns
        WHERE table_name = 'order_item' AND table_schema = (SELECT database()) ORDER BY ordinal_position
```
</br>

* <span style="color:orange">查询表的备注</span>
```sql
SHOW FULL COLUMNS FROM table_name;
```
</br>

* <span style="color:orange">查询表的DDL(字段类型, 备注)</span>

<img width="30%" src="/note/_v_images/code_sofa/sql/DDL.png" alt="DDL" />