## 1.引言
本文探讨两个问题：
* *如何在一个存储过程中使用从另一个存储过程返回的结果集？换句话说，如何在一个SELECT语句中使用从一个存储过程得到的结果集？*
* *如何通过传参，将数据表在存储过程之间传递*

在本文中我将会讨论解决上述问题的数种方法，并指出它们各自的优缺点。有些方法适用于从存储过程`读取`数据的场景，有些适合于给存储过程`传入`数据。也有一些方法既适合读取，也适合传入。当你想读取存储过程返回的结果集数据时，多数方法需要你重写被调用的存储过程，少数则不需要。

下表是我将要讨论的所有方法的汇总。*最低版本*指支持该方法的SQL Server的最低版本，空值代表该方法在SQL 2000以后的所有版本都可用。
<table>
  <tbody><tr><th>方法</th>
    <th>传入/读取</th>
    <th>是否须重写被调用sp?</th>
    <th>最低版本</th><th>备注</th></tr>
<tr class="tbltoplvl"><td><a href="#OUTPUT">
  <nomeddle>OUTPUT参数</nomeddle>
</a></td>
    <td>读取</td><td>是</td><td>&nbsp;</td>
    <td>不通用，但有时被低估了。</td></tr>
<tr class="tbltoplvl" style="border-bottom:none"><td><a href="#UDF">Table-valued Functions表值函数</a></td>
    <td rowspan="3">读取</td>
    <td rowspan="3">是</td>
    <td rowspan="3">&nbsp;</td>
    <td>通常是只读取数据时最好的方法，但有一些局限性。</td></tr>
<tr class="tblsublvl" style="padding-left:18px"><td>
    <a href="#inlineUDF">Inline Functions内嵌表值函数</a></td>
    <td>用于重用<span class="keyword">SELECT</span>语句。</td></tr>
<tr class="tblsublvl"><td>
    <a href="#multiUDF"><span class="nowrap">Multi-statement Functions多声明表值函数</span></a></td>
    <td>用于封装逻辑更为复杂的SQL语句</td></tr>
<tr class="tbltoplvl"><td><a href="#usingtable">Using a Table使用表</a></td>
    <td rowspan="3">传入/读取</td>
    <td rowspan="3">是的</td>
    <td rowspan="3">&nbsp;</td>
    <td>最为通用的方法。对于既需传入也需读取数据的场景，这是笔者最爱的选择。</td></tr>
<tr class="tblsublvl"><td><a href="#temptables">Sharing a Temp Table共享临时表</a></td>
    <td>主要用于调用者与被调用者是一对一关系的场景</td></tr>
<tr class="tblsublvl"><td>
    <a href="#prockeyed">Process-keyed Table</a></td>
    <td>是多个调用者调用一个存储过程的场景的最佳选择</td></tr>
<tr class="tbltoplvl">
  <td><a href="#tableparam">Table-valued Parameters表值参数</a></td>
    <td>传入</td><td>是</td><td>SQL&nbsp;2008</td>
    <td>通常用于客户端代码调用存储过程时传入数据</td></tr>
<tr class="tbltoplvl"><td><a href="#INSERTEXEC">
  <nomeddle>INSERT-EXEC语句</nomeddle>
</a></td>
    <td>读取</td><td><b>否</b></td><td>&nbsp;</td>
    <td>看似很有有吸引力的方法，但应尽量少用</td></tr>
<tr class="tbltoplvl"><td><a href="#CLR">
  <nomeddle>使用CLR</nomeddle>
</a></td>
    <td>读取</td><td><b>否</b></td><td>SQL&nbsp;2005</td>
    <td>复杂，但当INSERT-EXEC语句无法使用时，可作为最后的选择</td></tr>
<tr class="tbltoplvl"><td class="nomeddle"><a href="#OPENQUERY">
  <nomeddle>OPENQUERY</nomeddle>
</a></td>
    <td>读取</td><td><b>否</b></td><td>&nbsp;</td>
    <td>有许多坑，不建议使用</td></tr>
<tr class="tbltoplvl"><td><a href="#XML">
  <nomeddle>Using XML使用XML</nomeddle>
</a></td>
    <td>传入/读取</td><td>是</td><td>SQL&nbsp;2005</td>
    <td>不太规范的办法，但也有它的优势。</td></tr>
<tr class="tbltoplvl"><td><a href="#cursor">Using Cursor Variables使用游标变量</a></td>
    <td>读取</td><td>是</td><td>&nbsp;</td>
    <td>不推荐</td></tr>
<tr class="tbltoplvl">
  <td><a href="#sessioncontext">Session Context</a></td>
    <td>传入/读取</td>
    <td>是</td>
    <td>SQL&nbsp;2016</td>
    <td>不太常用，但可将数据保存在进程的全局变量中</td></tr>
</tbody></table>

在本文最后，我将会简要讨论一下存储过程位于不同服务器时的场景，这是一个相当有挑战的问题。

另一个相关的问题：如何从客户端代码给sp传入表数据，这个话题不在本文讨论范围之内，详见我的另一篇文章:[Using Table-Valued Parameters in SQL Server and .NET](http://www.sommarskog.se/arrays-in-sql-2008.html)。

本文有关**作者，标题，销量**的示例表数据运行在旧版示例数据库pubs。你可以在这里下载脚本[Microsoft's web site](https://www.microsoft.com/en-us/download/details.aspx?displaylang=en&id=23654)。(有一些例子使用完全不存在的表，所以在pubs里无法运行)
<span id="OUTPUT"><span>
## 2. OUTPUT参数
这一方法仅适用于返回结果集只有一行的情况，虽然如此，这个方法也有时被人所遗忘。假如有以下存储过程:
```
CREATE PROCEDURE insert_customer @name    nvarchar(50),
                                 @address nvarchar(50),
                                 @city    nvarchar(50) AS
DECLARE @cust_id int
BEGIN TRANSACTION
SELECT @cust_id = coalesce(MAX(cust_id), 0) + 1 FROM customers (UPDLOCK)
INSERT customers (cust_id, name, address, city)
   VALUES (@cust_id, @name, @address, @city)
COMMIT TRANSACTION
SELECT @cust_id
```
该存储过程往表中插入一条记录，并且返回这条记录的id。
重写该存储过程如下：
```
CREATE PROCEDURE insert_customer @name    nvarchar(50),
                                 @address nvarchar(50),
                                 @city    nvarchar(50),
                                 @cust_id int OUTPUT AS
BEGIN TRANSACTION
SELECT @cust_id = coalesce(MAX(cust_id), 0) + 1 FROM customers (UPDLOCK)
INSERT customers (cust_id, name, address, city)
   VALUES (@cust_id, @name, @address, @city)
COMMIT TRANSACTION
```
现在你可以直接在其他存储过程中调用`insert_customer`了。别忘了给调用T-SQL语句指定一个OUTPUT类型的参数。
```
EXEC insert_customer @name, @address, @city, @cust_id OUTPUT
```
**注意**：上面的例子只有一个output参数，其实一个存储过程可以有多个output参数。
## 3. Table-valued Functions表值函数
当你想复用一个存储过程返回的结果集时，首先要做的事情是去研究一下是否能将存储过程重写为一个表值函数。通常情况是不能，因为SQL Server对函数中的限制非常多。但如果能的话，那重写为表值函数就是最好的选择了。

SQL Server有两种表函数：行内和多行函数(inline and multi-statement functions)。
### 3.1 Inline Functions行内函数
以下是一个行内函数，摘自Books Online for SQL 2000。
```
CREATE FUNCTION SalesByStore (@storeid varchar(30))
RETURNS TABLE AS
RETURN (SELECT t.title, s.qty
        FROM   sales s
        JOIN   titles t ON t.title_id = s.title_id
        WHERE  s.stor_id = @storeid)
```
使用它，非常简单：
```
SELECT * FROM SalesByStore('6380')
```
可以在语句中使用where条件过滤数据，也可以加上其他表写更为复杂的查询语句。也就是说，可以将行内函数看作是一个表或者视图。你可以将一个行内函数视为一个带有参数的视图，因为查询优化器（query optimizer）把行内函数当做一个宏命令，生成执行计划，就和你用扩展查询语句一样。所以，将一个SELECT语句封装进行内函数是没有性能损耗的。也因此，当你需要复用只用一条SELECT语句的存储过程的时候，最好的办法就是将存储过程重写为一个行内函数（或者将SELECT语句写到行内函数中，再把行内函数包在你的存储过程中，这样调用客户端就不受任何影响了）。

有一些系统函数是没法写在用户定义函数（UDF）中的，比如newid()和rand()，SQL Server认为这么做会带来副作用。SQL 2000对此的限制更为严格的，行内函数里不能包含所有*非确定的*的函数。*非确定的*函数指对于同样的输入参数但是可能会得到不同的返回值的函数。一个典型的例子是getdate()。
### 3.2 Multi-statement Functions多声明表值函数
多声明表值函数可以包含任意多的语句。你需要声明一个返回表，然后将你要返回的数据插入到表中。上文的例子，用多声明表值函数实现如下：
```
CREATE FUNCTION SalesByStore (@storeid varchar(30))
   RETURNS @t TABLE (title varchar(80) NOT NULL PRIMARY KEY,
                     qty   smallint    NOT NULL)  AS
BEGIN
   INSERT @t (title, qty)
      SELECT t.title, s.qty
      FROM   sales s
      JOIN   titles t ON t.title_id = s.title_id
      WHERE  s.stor_id = @storeid
   RETURN
END
```
使用多声明表值函数的方式与行内函数一样。不同处在于，它不是在一个扩展查询语句中返回数据，而是像存储过程一样，用表变量将数据返回。这允许你将包含复杂语句的存储过程写到多声明表值函数中。
如上例所示，可以为返回表定义主键。我认为这绝对是一个最佳实践，有两个原因：
1. 它预定义了数据。如果你的预定义是不对的，你能及早发现，而不是在执行后花时间去查为何返回了错误了数据。
2. 当你在用函数返回大量数据时，主键对于优化器提升性能非常重要。
显而易见，这一优势仅当主键是定义在函数内部的列中时存在。给返回表加一个自增主键列是没有用的。

与行内函数比较，多声明表值函数因为返回表的原因，有一些劣势。更重要的是，当你在一个SQL语句中调用多声明表值函数并且将它与其他表进行关联的时候，优化器并不知道多声明表值函数的返回值的schema，所以它只能作出一种标准的判断，这导致在越大数据量的时候，优化器越可能作出不正确的估计，生成低效的执行计划(query plan)。解决这一问题的一个办法是将返回数据插入到一个临时表中，临时表的schema是预定义的，所以优化器能作出一个更优的执行计划。
> 注意：在SQL 2007中，微软给多声明表值函数新增了一个名叫隔行扫描执行的特性。当编译查询语句的时候，先运行函数，优化器从而可知函数返回多少行记录，基于此生成剩下的执行计划。但是因为没有分布统计，使用临时表的性能仍要更好一些。而且要知道隔行扫描执行只对SELECT语句有效，对SELECT INTO，INSERT，UPDATE和MERGE均无效。隔行扫描执行对SELECT INTO，INSERT无效颇有迷惑性，比如，你用SELECT语句没有问题，但是你想把SELECT数据放到一个表中，用了INSERT或SELECT INTO之后，隔行扫描执行却不起效了，SQL优化器便抓瞎了。

用户自定义函数（UDF）的限制相当多，譬如它不能改变数据库状态。以下列出UDF中一些最主要的限制：
* 只能在UDF中对本地表进行INSERT,UPDATE或DELETE的操作。
* 不能调用存储过程（除了扩展存储过程）。
* 不能执行动态SQL。
* 只能使用但不能创建表，包括实体表和临时表。
* 不能使用RAISERROR, TRY-CATCH和BEGIN/COMMIT/ROLLBACK TRANSACTION等命令。
* 不能使用"副作用(side-effecting)"系统函数，比如newid()和rand()。
* SQL2000不能使用结果非确定的系统函数。
请查阅Books Online上关于[CREATE FUNCTION](https://docs.microsoft.com/en-us/sql/t-sql/statements/create-function-transact-sql)的讨论文章中Remarks部分的内容。
## 4. Using a Table使用表
在数据库传递数据，还有啥能比使用表更好的办法呢？使用表可没有上述在使用函数时的那些条条框框限制。使用表主要有两种方法：1）共享一个本地临时表。2）使用一个process-keyed table。前者更为轻量，但它有一个维护性问题——它不固定的字段；而后者通过使用一个固定的表结构从而没有这个维护性问题。但是这两个方法存在再编译的问题，尤其是对于被调用频率非常高的存储过程，这一问题尤为严重。process-keyed table方法的问题严重性稍微低些。使用本地临时表也会带来本地缓存垃圾的风险。我会在下文中详述这些问题。
### 4.1 Sharing a Temp Table共享临时表
#### 简述
这一方法本身非常简单：
```
CREATE PROCEDURE inner_sp @par1 int,
                          @par2 bit,
                          ... AS
   ...
   INSERT/UPDATE/DELETE #tmp
go
CREATE PROCEDURE outer_sp AS
   DECLARE ...
   CREATE TABLE #mytemp (col1 int     NOT NULL,
                         col2 char(5) NULL,
                        ...)
   ...
   EXEC inner_sp @par1, @par2 ...
   SELECT * FROM #mytemp
go
```
本例中，**outer\_sp**创建临时表，**inner\_sp**往临时表写入数据，这是临时表作为输出的应用场景。另一个场景是，**outer\_sp**往临时表写入数据，然后**inner\_sp**对数据进行一些通用的计算处理，调用者**outer\_sp**再使用处理过后的数据再进行其他操作。这一场景就是将临时表同时用作输入与输出。还有一种就是调用者在临时表中写入数据，被调用者对数据进行一系列业务逻辑验证，然后再对数据进行更新。这就是临时表作为输入时的场景。
#### 改动现有代码
假设你有如下存储过程:
```
CREATE PROCEDURE SalesByStore @storeid varchar(30) AS
   SELECT t.title, s.qty
   FROM   sales s
   JOIN   titles t ON t.title_id = s.title_id
   WHERE  s.stor_id = @storeid
```
你想要在另一个存储过程中复用该存储过程返回的结果集，但你只想要特定销量以上的那些书的标题，你该如何用临时表完成这个需求的同时不影响现有的其他客户端？解决办法是将这个存储过程放到一个子过程里，然后在子过程上再封装一层存储过程：
```
CREATE PROCEDURE SalesByStore_core @storeid varchar(30) AS
   INSERT #SalesByStore (title, qty)
      SELECT t.title, s.qty
      FROM   sales s
      JOIN   titles t ON t.title_id = s.title_id
      WHERE  s.stor_id = @storeid
go
CREATE PROCEDURE SalesByStore @storeid varchar(30) AS
   CREATE TABLE #SalesByStore(title varchar(80) NOT NULL PRIMARY KEY,
                              qty   smallint    NOT NULL)
   EXEC SalesByStore_core @storeid
   SELECT * FROM #SalesByStore
go
CREATE PROCEDURE BigSalesByStore @storeid varchar(30),
                                 @qty     smallint AS
   CREATE TABLE #SalesByStore(title varchar(80) NOT NULL PRIMARY KEY,
                              qty   smallint    NOT NULL)
   EXEC SalesByStore_core @storeid
   SELECT * FROM #SalesByStore WHERE qty >= @qty
go
EXEC SalesByStore '7131'
EXEC BigSalesByStore '7131', 25
go
DROP PROCEDURE SalesByStore, BigSalesByStore, SalesByStore_core
```
**注意**：这段脚本完整地创建，测试并删除对象，在下文中还将用到这些存储过程脚本。
与多声明表值函数的例子一样，我出于相同的目的，也给临时表加了主键。有些读者可能会有疑虑，从最佳实践的角度说，用SELECT *并不是一个好的做法。但我认为在同一个存储过程中创建的临时表身上用SELECT *是没有问题的，特别是当你需要获取临时表的所有列的时候。(当SELECT *的表对象是在别处定义的时候，情况就不一样了，因为你可能没法知道表在别处是否被改动了)

解决方法非常明确，但是你可能会觉得在两个地方创建了临时表不太优雅，并且还多加了一个存储过程。那么下面这个稍显的方法能避免这些问题：
```
CREATE PROCEDURE SalesByStore_core @storeid       varchar(30),
                                   @wantresultset bit = 0 AS
   IF object_id('tempdb..#SalesByStore') IS  NULL 
   BEGIN
      CREATE TABLE #SalesByStore(title varchar(80) NOT NULL PRIMARY KEY,
                                 qty   smallint    NOT NULL)
   END

   INSERT #SalesByStore (title, qty)
      SELECT t.title, s.qty
      FROM   sales s
      JOIN   titles t ON t.title_id = s.title_id
      WHERE  s.stor_id = @storeid
      
   IF @wantresultset = 1
      SELECT * FROM #SalesByStore
go
CREATE PROCEDURE SalesByStore @storeid varchar(30) AS
   EXEC SalesByStore_core @storeid, 1
go
```
我将创建临时表从封装sp挪到了子过程，子过程检测临时表是否存在，不存在的话会创建临时表。外层sp中有一个EXEC语句，并给**@wantresultset**传参"1"去获取数据。如果不传此参数，则它默认为0，调用**BigSalesByStore**则不受新的逻辑的影响。（因此，这里仍存在两处CREATE TABLE语句）

#### 代码复用的优点
在我们继续之前，我想说明上面的例子并不是一个好的实践。共享临时表方法本身并不是个坏的方法，而是说你需要根据场景有选择性地使用这个方法。就比如上面这个例子，你可能也意识到了，我们又创建临时表，又是创建新的存储过程，却只为解决一个如此简单的问题，可谓是杀鸡用牛刀。共享临时表的方法在代码量比较大的时候，就是一个非常好的方法了。上面例子只是为了为讲清楚方法而简化了场景。

要记住，与C#和JAVA之类的语言相比，Transact-SQL的代码复用特性非常简陋，所以我们在尝试复用代码的时候的方法显得异常繁琐。因此，在T‑SQL中进行代码复用要特别注意使用场景。代码复用依然是有它的优势，只不过在T‑SQL中并不像现代面向对象语言那么明显。在本文例子的简单问题中，最好的解法显然是给**SalesByStore**加上一个参数@qty。如果出于一些原因不能加参数，那么复制一个**BigSalesByStore**存储过程也比共享临时表更好一些。

#### 性能影响
你需要知道共享临时表存在两个性能方面的问题：一是它会产生相当数量的缓存垃圾，二是里层存储过程中与共享临时表相关的所有语句在每次执行时都会被重新编译。

许多年来我已经成功让自己忽视了第一个问题，然而Alex Friedman却又让我重视起这一问题。在官方文档[Plan Caching in SQL Server 2008](https://technet.microsoft.com/en-us/library/ee343986)(虽然标题2008，但也适用后续版本)中是这么说的：
> 如果一个存储过程使用了并不是由它静态地创建的临时表，那么spid(进程ID)会被存入缓存键。这意味着该存储过程的执行计划只会被在同一session中共享复用。如果临时表是由该存储过程静态创建的，那么则不会有这个问题。

也就是说，如果两个sessions同时调用**outer\_sp**，那么他们各自在缓存中都有一个**inner\_sp**缓存项。假设你有一个繁忙的系统，有上千个sessions执行了**outer\_sp**，那么光这一个存储过程就有上千条缓存项。想象一下，如果你在10-20处都用了这个方法，那么系统中的缓存项高达20000。这是一个相当大的数量。由此带来的另一个后果就是，因为[参数探测](http://www.sommarskog.se/query-plan-mysteries.html)的缘故，不同的session id对应着不同的执行计划，这可能会给你带来一些困惑。

有一个方法可以避免缓存垃圾，当然它也有一定的代价：给内层存储过程增加WITH RECOMPILE。这会防止该存储过程的执行计划被写入缓存，但是这也意味着这个存储过程每次执行都会被编译。但这也比之前好一些，因为即便不使用WITH RECOMPILE，也会遇到许多内层存储过程重新编译的性能问题。

**outer\_sp**每次被调用，**inner\_sp**都会生成共享临时表的新实例。没有任何能保证临时表的schema与上次一样，所以，SQL Server会重新编译**inner\_sp**中所以与共享临时表相关的语句。只要存储过程被调用的频率不是很高，重新编译就不是太需要考虑的问题，但是如果是高频调用的场景，这会导致CPU使用率升高的问题。

如果**outer\_sp**被调用的非常频繁，但是session数量不太大，你也不必使用WITH RECOMPILE去规避缓存垃圾的问题。如果共享临时表的数据量不是很大，你可以用这个办法降低重新编译次数：将**inner\_sp**中的临时表在存储过程中静态定义(这也意味着你不能用SELECT INTO创建临时表)。在存储过程的开始和结束处读取或写入数据到临时表中。为保万无一失，我认为你应该在临时表有关的语句里使用OPTION (KEEPFIXED PLAN)命令，这样能防止因statistics变化而导致的临时表重新编译。
#### 维护问题
如果内层存储过程在多处被调用，这时你要想修改内层存储过程返回或写入的字段，那么你就得把所有调用它的存储过程中的临时表定义改一遍。因此，共享临时表通常用于调用与被调用存储过程一一的简单对应的场景。或者，你的临时表字段非常少，比如只有一个字段——客户IDs，通常不会轻易改动。

有一些方法可以规避这个维护问题。一个方法是使用process-keyed table,我们将在下一小节详述。我从本文的读者也收到了不少很有趣的点子。

其中一个点子来自Richard St-Aubin。调用方创建一个临时表，只有一个作为摆设的字段。然后再调用一个存储过程用ALTER TABLE去给临时表增加真正我们需要的字段。用例子说明就像这样：
```
CREATE PROCEDURE inner_sp @par1 int,
                          @par2 bit,
                          ... AS
   ...
   INSERT/UPDATE/DELETE #mytemp
go
CREATE PROCEDURE define_temp_table AS
   ALTER TABLE #mytemp ADD col1 int     NOT NULL,
                           col2 char(5) NULL,
                           ...
go
CREATE PROCEDURE outer_sp AS
   DECLARE ...
   CREATE TABLE #mytemp (dummycol bit)
   EXEC define_temp_table
   ...
   EXEC inner_sp @par1, @par2 ...
   SELECT * FROM #mytemp
go
```
你必须在**outer\_sp**中创建临时表，因为如果在**define\_temp\_table**中创建的话，那么临时表在**define\_temp\_table**结束的时候就将被删除了。这个方法的确值得考虑，但这个方法存在前文讨论过的字段引起的重新编译的问题。而且这个方法的临时表没有缓存。在高频调用的时候，这一点可能会明显地影响性能。

另外一个方法来自Wayne Bloss。这个方法要求SQL 2008或以上版本。它创建了一个table type，存入了临时表的字段定义。你可能只用table type来声明表变量和表参数，Wayne会这么用：
```
DECLARE @dummy my_table_type
SELECT * INTO #mytemp FROM @dummy
```
之后你只和**#mytemp**交互；**@dummy**的唯一作用为临时表**#mytemp**准备了一个预定义(如果你不熟悉table types,我们会在下文的table-valued parameters章节进行详细讨论)。这个方法的一大局限性是，你只能定义字段，而不能保留constraints，因为constraints并不会随着SELECT INTO复制到临时表。你可能会想，很少会给临时表加constraints吧。但我发现，给临时表加constraints非常有用，它帮助SQL去判断预估临时表的数据。这当然不止适用于存储过程间共享的临时表。给你的临时表加上主键，也有助于临时表在join时的性能。

在本小节最后，我想指出这种共享临时表的方法非常灵活。被调用的存储过程只管读写字段，调用方可根据它的需要在创建临时表的时候增加字段。因而，两个存储过程调用同一个存储过程的时候可以分别用不同定义的临时表，只要被调用的存储过程用到的字段有定义即可。
#### 关于SQL Server数据工具
SQL Server数据工具，简称SSDT，有诸多用处，能给你带来许多便利。比如如果你想了一个存储过程：
```
CREATE PROCEDURE example_sp AS
   CREATE TABLE #temp(a int NOT NULL)
   SELECT a FROM #temmp
```
SSDT会在你执行语句之前就能告诉你临时表的名字拼写错了。这个特性非常有用，能帮助你提早发现错误。但是SSDT不认识共享临时表，所有SSDT会对**SalesByStore\_core**这样存储过程报警告，准确说会报三个警告，每一个字段一个。虽然只是警告，代码可以继续运行，但是存储过程数量稍微多一些，这些警告填满错误列表窗口，有可能使你漏掉了真正重要的警告或者问题提示。

有一个办法可以屏蔽警告：右键点击Solution Explorer中的文件，选择属性。有一个属性叫*Suppress T‑Sql Warning*，输入要屏蔽的错误的代码。但是这样做会使得SSDT不去检查存储过程中的所有表名称。没有办法只针对共享临时表进行设置。

言而总之，如果你使用SSDT，你会发现SSDT对共享临时表并不友好。
### 4.2 Process-Keyed Tables
这个方法使用普通表，从而规避了缓存垃圾问题和维护问题。但重新编译的问题仍旧存在，只是它的本质是不一样的。
#### 梗概
Process-Keyed Table指的是那些被用作临时表的普通表。为了让多个进程同时使用表，表中有一个特别的字段来标识进程。最简单的方法就是全局变量**@@spid**（**@@spid**是SQL Server的进程id）。这个方法其实非常常用，所以Process-Keyed Table又被称为*spid-keyed tables*。这里是简例，后文我将会再举一个更为完整的例子。
```
CREATE TABLE process_keyed (spid  int     NOT NULL,
                            col1  int     NOT NULL,
                            col2  char(5) NULL,
                            ...)
go
CREATE CLUSTERED INDEX processkey_ix ON process_keyed (spid)
-- Add other columns as needed.
go
...
DELETE process_keyed WHERE spid = @@spid
INSERT process_keyed (spid, col1, col2, ....)
   VALUES (@@spid, @val1, @val2, ...)
...
SELECT col1, col2, ...
FROM   process_keyed
WHERE  spid = @@spid
...
DELETE process_keyed WHERE spid = @@spid
```
一些需要注意的：
1. 表应该给进程字段（比如本例中的**spid**）加上聚集索引，所有与表相关的查询均包括查询条件`WHERE spid = @@spid`。
2. 对于某个**@@spid**，在你给表插入数据之前，删除所有这个**s@@spid**下的数据，以保证安全。
3. 使用完的数据应立刻删除，以免数据占据存储空间。
#### 选择进程键
虽然通常会使用**@@spid**作为表中的进程字段，但是这存在两个问题：
1. 一旦哪个粗心的程序员忘了在使用前后删除数据，老数据可能会被存储过程读取，从而会导致错误的结果。而且这个错误难以被发现。
2. 一个客户端传参**@@spid**，但并不能保证它每次传的都是同一个**@@spid**。