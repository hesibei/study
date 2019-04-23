<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Title</title>
</head>
<body>
<div id="cnblogs_post_body" class="blogpost-body cnblogs-markdown"><p>摘要</p>
    <blockquote>
        <p><strong>企业千万家，靠谱没几家。<br>
            社招选错家，亲人两行泪。</strong></p>
    </blockquote>
    <p>祝大家金三银四跳槽顺利！</p>
    <h2 id="引言">引言</h2>
    <p>开始我们的内容，相信大家一定遇到过下面的一个面试场景</p>
    <blockquote>
        <p><strong>面试官：“讲讲mysql有几个事务隔离级别？”<br>
            你：“读未提交，读已提交，可重复读，串行化四个！默认是可重复读”<br>
            面试官：“为什么mysql选可重复读作为默认的隔离级别？”<br>
            (你面露苦色，不知如何回答！)<br>
            面试官:"你们项目中选了哪个隔离级别？为什么？"<br>
            你：“当然是默认的可重复读，至于原因。。呃。。。”<br>
            (然后你就可以回去等通知了！)</strong></p>
    </blockquote>
    <p>为了避免上述尴尬的场景，请继续往下阅读！<br>
        Mysql默认的事务隔离级别是<strong>可重复读(Repeatable Read)</strong>，那互联网项目中Mysql也是用默认隔离级别，不做修改么？<br>
        OK，不是的，我们在项目中一般用<strong>读已提交(Read Commited)</strong>这个隔离级别！<br>
        what！居然是读已提交，网上不是说这个隔离级别存在<code>不可重复读</code>和<code>幻读</code>问题么？不用管么？好，带着我们的疑问开始本文！</p>
    <h2 id="正文">正文</h2>
    <p>我们先来思考一个问题，在Oracle，SqlServer中都是选择<strong>读已提交(Read Commited)</strong>作为默认的隔离级别，为什么Mysql不选择<strong>读已提交(Read Commited)</strong>作为默认隔离级别，而选择<strong>可重复读(Repeatable Read)</strong>作为默认的隔离级别呢？<br>
        <img src="https://img2018.cnblogs.com/blog/725429/201903/725429-20190311134825464-1181946735.jpg"></p>
    <h3 id="whywhywhy">Why?Why?Why?</h3>
    <p>这个是有历史原因的，当然要从我们的主从复制开始讲起了！<br>
        <em>主从复制，是基于什么复制的？</em><br>
        是基于binlog复制的！这里不想去搬binlog的概念了，就简单理解为binlog是一个记录数据库更改的文件吧～<br>
        <em>binlog有几种格式？</em><br>
        OK，三种，分别是</p>
    <ul>
        <li>statement:记录的是修改SQL语句</li>
        <li>row：记录的是每行实际数据的变更</li>
        <li>mixed：statement和row模式的混合</li>
    </ul>
    <p>那Mysql在5.0这个版本以前，binlog只支持<code>STATEMENT</code>这种格式！而这种格式在<strong>读已提交(Read Commited)</strong>这个隔离级别下主从复制是有bug的，因此Mysql将<strong>可重复读(Repeatable Read)</strong>作为默认的隔离级别！<br>
        接下来，就要说说当binlog为<code>STATEMENT</code>格式，且隔离级别为<strong>读已提交(Read Commited)</strong>时，有什么bug呢？如下图所示，在主(master)上执行如下事务<br>
        <img src="https://img2018.cnblogs.com/blog/725429/201903/725429-20190311134942591-1582271936.jpg"><br>
        此时在主(master)上执行下列语句</p>
    <pre><code class="hljs sql"><span class="hljs-keyword">select</span> * <span class="hljs-keyword">from</span> <span class="hljs-keyword">test</span>；</code></pre>
    <p>输出如下</p>
    <pre><code class="hljs lua">+<span class="hljs-comment">---+</span>
| b |
+<span class="hljs-comment">---+</span>
| <span class="hljs-number">3</span> |
+<span class="hljs-comment">---+</span>
<span class="hljs-number">1</span> row <span class="hljs-keyword">in</span> set</code></pre>
    <p>但是，你在此时在从(slave)上执行该语句，得出输出如下</p>
    <pre><code class="hljs vbscript"><span class="hljs-literal">Empty</span> <span class="hljs-keyword">set</span></code></pre>
    <p>这样，你就出现了主从不一致性的问题！原因其实很简单，就是在master上执行的顺序为先删后插！而此时binlog为STATEMENT格式，它记录的顺序为先插后删！从(slave)同步的是binglog，因此从机执行的顺序和主机不一致！就会出现主从不一致！<br>
        <em>如何解决？</em><br>
        解决方案有两种！<br>
        (1)隔离级别设为<strong>可重复读(Repeatable Read)</strong>,在该隔离级别下引入间隙锁。当<code>Session 1</code>执行delete语句时，会锁住间隙。那么，<code>Ssession 2</code>执行插入语句就会阻塞住！<br>
        (2)将binglog的格式修改为row格式，此时是基于行的复制，自然就不会出现sql执行顺序不一样的问题！奈何这个格式在mysql5.1版本开始才引入。因此由于历史原因，mysql将默认的隔离级别设为<strong>可重复读(Repeatable Read)</strong>，保证主从复制不出问题！</p>
    <p>那么，当我们了解完mysql选<strong>可重复读(Repeatable Read)</strong>作为默认隔离级别的原因后，接下来我们将其和<strong>读已提交(Read Commited)</strong>进行对比，来说明为什么在互联网项目为什么将隔离级别设为<strong>读已提交(Read Commited)</strong>！</p>
    <h3 id="对比">对比</h3>
    <p>ok，我们先明白一点！项目中是不用<strong>读未提交(Read UnCommitted)</strong>和<strong>串行化(Serializable)</strong>两个隔离级别，原因有二</p>
    <ul>
        <li>采用<strong>读未提交(Read UnCommitted)</strong>,一个事务读到另一个事务未提交读数据，这个不用多说吧，从逻辑上都说不过去！</li>
        <li>采用<strong>串行化(Serializable)</strong>，每个次读操作都会加锁，快照读失效，一般是使用mysql自带分布式事务功能时才使用该隔离级别！(笔者从未用过mysql自带的这个功能，因为这是XA事务，是强一致性事务，性能不佳！互联网的分布式方案，多采用最终一致性的事务解决方案！)</li>
    </ul>
    <p>也就是说，我们该纠结都只有一个问题，究竟隔离级别是用读已经提交呢还是可重复读？<br>
        接下来对这两种级别进行对比，讲讲我们为什么选<strong>读已提交(Read Commited)</strong>作为事务隔离级别！<br>
        假设表结构如下</p>
    <pre><code class="hljs sql"> <span class="hljs-keyword">CREATE</span> <span class="hljs-keyword">TABLE</span> <span class="hljs-string">`test`</span> (
<span class="hljs-string">`id`</span> <span class="hljs-built_in">int</span>(<span class="hljs-number">11</span>) <span class="hljs-keyword">NOT</span> <span class="hljs-literal">NULL</span>,
<span class="hljs-string">`color`</span> <span class="hljs-built_in">varchar</span>(<span class="hljs-number">20</span>) <span class="hljs-keyword">NOT</span> <span class="hljs-literal">NULL</span>,
PRIMARY <span class="hljs-keyword">KEY</span> (<span class="hljs-string">`id`</span>)
) <span class="hljs-keyword">ENGINE</span>=<span class="hljs-keyword">InnoDB</span></code></pre>
    <p>数据如下</p>
    <pre><code class="hljs smalltalk">+----+-------+
| id | color |
+----+-------+
|  <span class="hljs-number">1</span> |  red  |
|  <span class="hljs-number">2</span> | white |
|  <span class="hljs-number">5</span> |  red  |
|  <span class="hljs-number">7</span> | white |
+----+-------+</code></pre>
    <p>为了便于描述，下面将</p>
    <ul>
        <li><strong>可重复读(Repeatable Read)</strong>，简称为RR；</li>
        <li><strong>读已提交(Read Commited)</strong>，简称为RC；</li>
    </ul>
    <p><em>缘由一：在RR隔离级别下，存在间隙锁，导致出现死锁的几率比RC大的多！</em><br>
        此时执行语句</p>
    <pre><code class="hljs sql"><span class="hljs-keyword">select</span> * <span class="hljs-keyword">from</span> <span class="hljs-keyword">test</span> <span class="hljs-keyword">where</span> <span class="hljs-keyword">id</span> &lt;<span class="hljs-number">3</span> <span class="hljs-keyword">for</span> <span class="hljs-keyword">update</span>;</code></pre>
    <p>在RR隔离级别下，存在间隙锁，可以锁住(2,5)这个间隙，防止其他事务插入数据！<br>
        而在RC隔离级别下，不存在间隙锁，其他事务是可以插入数据！</p>
    <p><code>ps</code>:在RC隔离级别下并不是不会出现死锁，只是出现几率比RR低而已！</p>
    <p><em>缘由二：在RR隔离级别下，条件列未命中索引会锁表！而在RC隔离级别下，只锁行</em><br>
        此时执行语句</p>
    <pre><code class="hljs sql"><span class="hljs-keyword">update</span> <span class="hljs-keyword">test</span> <span class="hljs-keyword">set</span> color = <span class="hljs-string">'blue'</span> <span class="hljs-keyword">where</span> color = <span class="hljs-string">'white'</span>; </code></pre>
    <p>在RC隔离级别下，其先走聚簇索引，进行全部扫描。加锁如下：<br>
        <img src="https://img2018.cnblogs.com/blog/725429/201903/725429-20190311135044150-2146259021.png"><br>
        但在实际中，MySQL做了优化，在MySQL Server过滤条件，发现不满足后，会调用unlock_row方法，把不满足条件的记录放锁。<br>
        实际加锁如下<br>
        <img src="https://img2018.cnblogs.com/blog/725429/201903/725429-20190311135057008-2073392330.png"><br>
        然而，在RR隔离级别下，走聚簇索引，进行全部扫描，最后会将整个表锁上，如下所示<br>
        <img src="https://img2018.cnblogs.com/blog/725429/201903/725429-20190311135808965-164228503.jpg"></p>
    <p><em>缘由三：在RC隔离级别下，半一致性读(semi-consistent)特性增加了update操作的并发性！</em><br>
        在5.1.15的时候，innodb引入了一个概念叫做“semi-consistent”，减少了更新同一行记录时的冲突，减少锁等待。<br>
        所谓半一致性读就是，一个update语句，如果读到一行已经加锁的记录，此时InnoDB返回记录最近提交的版本，由MySQL上层判断此版本是否满足update的where条件。若满足(需要更新)，则MySQL会重新发起一次读操作，此时会读取行的最新版本(并加锁)！<br>
        具体表现如下:<br>
        此时有两个Session，Session1和Session2！<br>
        Session1执行</p>
    <pre><code class="hljs sql"><span class="hljs-keyword">update</span> <span class="hljs-keyword">test</span> <span class="hljs-keyword">set</span> color = <span class="hljs-string">'blue'</span> <span class="hljs-keyword">where</span> color = <span class="hljs-string">'red'</span>; </code></pre>
    <p>先不Commit事务！<br>
        与此同时Ssession2执行</p>
    <pre><code class="hljs sql"><span class="hljs-keyword">update</span> <span class="hljs-keyword">test</span> <span class="hljs-keyword">set</span> color = <span class="hljs-string">'blue'</span> <span class="hljs-keyword">where</span> color = <span class="hljs-string">'white'</span>; </code></pre>
    <p>session 2尝试加锁的时候，发现行上已经存在锁，InnoDB会开启semi-consistent read，返回最新的committed版本(1,red),(2，white),(5,red),(7,white)。MySQL会重新发起一次读操作，此时会读取行的最新版本(并加锁)!<br>
        而在RR隔离级别下，Session2只能等待！</p>
    <h3 id="两个疑问">两个疑问</h3>
    <p><em>在RC级别下，不可重复读问题需要解决么？</em><br>
        不用解决，这个问题是可以接受的！毕竟你数据都已经提交了，读出来本身就没有太大问题！Oracle的默认隔离级别就是RC，你们改过Oracle的默认隔离级别么？</p>
    <p><em>在RC级别下，主从复制用什么binlog格式？</em><br>
        OK,在该隔离级别下，用的binlog为row格式，是基于行的复制！Innodb的创始人也是建议binlog使用该格式！</p>
    <h2 id="总结">总结</h2>
    <p>本文啰里八嗦了一篇文章只是为了说明一件事，互联网项目请用：读已提交(Read Commited)这个隔离级别！</p>
</div>
</body>
</html>