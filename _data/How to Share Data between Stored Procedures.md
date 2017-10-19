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
### 3.2 Multi-statement Functions
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
> 注意：在SQL 2007中，微软给多声明表值函数新增了一个名叫隔行扫描执行的特性。当编译查询语句的时候，先运行函数，优化器从而可知函数返回多少行记录，基于此生成剩下的执行计划。但是因为没有分布统计，使用临时表的性能仍要更好一些。