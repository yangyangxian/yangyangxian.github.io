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
## 2. OUTPUT参数