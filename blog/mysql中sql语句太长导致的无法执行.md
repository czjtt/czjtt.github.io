问题：mysql中sql查询语句太长，导致的无法正常执行问题

解决：

设置group_concat的长度

SET GLOBAL group_concat_max_len=102400;
SET SESSION group_concat_max_len=102400;

根据配置文件会限制Server接受的数据包大小 

set GLOBAL max_allowed_packet=1073741824;
SET SESSION max_allowed_packet = 1073741824;



可以使用

```sql
show VARIABLES like 'max_allowed_packet';
```

查看设置的结果。

show variables 等同于 show session variables 

可以使用 show global varibales查询全局设置的结果。