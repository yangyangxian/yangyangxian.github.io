<style>
.tblsublvl {
    padding-left: 18px;
    border: 1px solid grey;
}
</style>

## 1.引言
本文探讨两个问题：
* 如何在一个存储过程中使用从另一个存储过程返回的结果集？换句话说，如何在一个SELECT语句中使用从一个存储过程得到的结果集？
* 如何通过传参，将数据表在存储过程之间传递？

在本文中我将会讨论解决上述问题的数种方法，并指出它们各自的优缺点。有些方法适用于从存储过程取数据的场景，有些适合于给存储过程传数据。也有一些方法既适合读取，也适合传入。当你想读取存储过程返回的结果集数据时，多数方法需要你重写被调用的存储过程，少数则不需要。

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
    <td>通常是只读取数据时最好的方法，但有一些局限。</td></tr>
<tr class="tblsublvl" style="padding-left:18pt;border:1px solid grey"><td>
    <a href="#inlineUDF">Inline Functions内嵌表值函数</a></td>
    <td>用于重用<span class="keyword">SELECT</span>语句.</td></tr>
<tr class="tblsublvl"><td>
    <a href="#multiUDF"><span class="nowrap">Multi-statement Functions多声明表值函数</span></a></td>
    <td>用于更为复杂逻辑的SQL语句When you need to encapsulate more complex
    logic.</td></tr>
<tr class="tbltoplvl"><td><a href="#usingtable">Using a Table</a></td>
    <td rowspan="3">In/Out</td>
    <td rowspan="3">Yes</td>
    <td rowspan="3">&nbsp;</td>
    <td>The most general solution. My favoured choice for input/output scenarios.</td></tr>
<tr class="tblsublvl"><td><a href="#temptables">Sharing a Temp Table</a></td>
    <td>Mainly for a single pair of caller/callee.</td></tr>
<tr class="tblsublvl"><td>
    <a href="#prockeyed">Process-keyed Table</a></td>
    <td>Best choice for many callers to the same callee.</td></tr>
<tr class="tbltoplvl">
  <td><a href="#tableparam">Table-valued Parameters</a></td>
    <td>Input</td><td>Yes</td><td>SQL&nbsp;2008</td>
    <td>Mainly useful when passing data from a client.</td></tr>
<tr class="tbltoplvl"><td><a href="#INSERTEXEC">
  <nomeddle>INSERT-EXEC</nomeddle>
</a></td>
    <td>Output</td><td><b>No</b></td><td>&nbsp;</td>
    <td>Deceivingly appealing, but should be used sparingly.</td></tr>
<tr class="tbltoplvl"><td><a href="#CLR">
  <nomeddle>Using the CLR</nomeddle>
</a></td>
    <td>Output</td><td><b>No</b></td><td>SQL&nbsp;2005</td>
    <td>Complex, but useful as a last resort when <span class="keyword">INSERT-EXEC</span> does not work.</td></tr>
<tr class="tbltoplvl"><td class="nomeddle"><a href="#OPENQUERY">
  <nomeddle>OPENQUERY</nomeddle>
</a></td>
    <td>Output</td><td><b>No</b></td><td>&nbsp;</td>
    <td>Tricky with many pitfalls. Discouraged.</td></tr>
<tr class="tbltoplvl"><td><a href="#XML">
  <nomeddle>Using XML</nomeddle>
</a></td>
    <td>In/Out</td><td>Yes</td><td>SQL&nbsp;2005</td>
    <td>A bit of a kludge, but not without advantages.</td></tr>
<tr class="tbltoplvl"><td><a href="#cursor">Using Cursor Variables</a></td>
    <td>Output</td><td>Yes</td><td>&nbsp;</td>
    <td>Not recommendable.</td></tr>
<tr class="tbltoplvl">
  <td><a href="#sessioncontext">Session Context</a></td>
    <td>In/Out</td>
    <td>Yes</td>
    <td>SQL&nbsp;2016</td>
    <td>Not a general method, but useful to keep data globally available in a process.</td></tr>
</tbody></table>


