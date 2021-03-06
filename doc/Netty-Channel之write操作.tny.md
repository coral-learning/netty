<div class="article-inner">


<header class="article-header">


<h1 class="article-title" itemprop="name">
精尽 Netty 源码解析 —— Channel（三）之 read 操作
</h1>


</header>

<div class="article-entry" itemprop="articleBody">

<!-- Table of Contents -->

<h1 id="1-概述"><a href="#1-概述" class="headerlink" title="1. 概述"></a>1. 概述</h1><p>本文分享 Netty NIO 服务端读取( <strong>read</strong> )来自客户端数据的过程、和 Netty NIO 客户端接收( <strong>read</strong> )来自服务端数据的结果。实际上，这两者的实现逻辑是一致的：</p>
<ul>
<li>客户端就不用说了，自身就使用了 Netty NioSocketChannel 。</li>
<li>服务端在接受客户端连接请求后，会创建客户端对应的 Netty NioSocketChannel 。</li>
</ul>
<p>因此，我们统一叫做 NioSocketChannel 读取( <strong>read</strong> )对端的数据的过程。</p>
<hr>
<p>NioSocketChannel 读取( <strong>read</strong> )对端的数据的过程，简单来说：</p>
<ol>
<li>NioSocketChannel 所在的 EventLoop 线程轮询是否有新的数据写入。</li>
<li>当轮询到有新的数据写入，NioSocketChannel 读取数据，并提交到 pipeline 中进行处理。</li>
</ol>
<p>比较简单，和 <a href="http://svip.iocoder.cn/Netty/Channel-2-accept">《精尽 Netty 源码解析 —— Channel（二）之 accept 操作》</a> 有几分相似。或者我们可以说：</p>
<ul>
<li>NioServerSocketChannel 读取新的连接。</li>
<li>NioSocketChannel 读取新的数据。</li>
</ul>
<h1 id="2-NioByteUnsafe-read"><a href="#2-NioByteUnsafe-read" class="headerlink" title="2. NioByteUnsafe#read"></a>2. NioByteUnsafe#read</h1><p>NioByteUnsafe ，实现 AbstractNioUnsafe 抽象类，AbstractNioByteChannel 的 Unsafe 实现类。代码如下：</p>
<figure class="highlight java"><table><tbody><tr><td class="code"><pre><span class="line"><span class="keyword">protected</span> <span class="class"><span class="keyword">class</span> <span class="title">NioByteUnsafe</span> <span class="keyword">extends</span> <span class="title">AbstractNioUnsafe</span> </span>{</span><br><span class="line"></span><br><span class="line">    <span class="function"><span class="keyword">public</span> <span class="keyword">final</span> <span class="keyword">void</span> <span class="title">read</span><span class="params">()</span> </span>{ <span class="comment">/** 省略内部实现 **/</span> }</span><br><span class="line"></span><br><span class="line">    <span class="function"><span class="keyword">private</span> <span class="keyword">void</span> <span class="title">handleReadException</span><span class="params">(ChannelPipeline pipeline, ByteBuf byteBuf, Throwable cause, <span class="keyword">boolean</span> close, RecvByteBufAllocator.Handle allocHandle)</span> </span>{ <span class="comment">/** 省略内部实现 **/</span> }</span><br><span class="line"></span><br><span class="line">    <span class="function"><span class="keyword">private</span> <span class="keyword">void</span> <span class="title">closeOnRead</span><span class="params">(ChannelPipeline pipeline)</span> </span>{ <span class="comment">/** 省略内部实现 **/</span> }</span><br><span class="line"></span><br><span class="line">}</span><br></pre></td></tr></tbody></table></figure>
<ul>
<li>一共有 3 个方法。但是实现上，入口为 <code>#read()</code> 方法，而另外 2 个方法被它所调用。所以，我们赶紧开始 <code>#read()</code> 方法的理解吧。</li>
</ul>
<h2 id="2-1-read"><a href="#2-1-read" class="headerlink" title="2.1 read"></a>2.1 read</h2><p>在 NioEventLoop 的 <code>#processSelectedKey(SelectionKey k, AbstractNioChannel ch)</code> 方法中，我们会看到这样一段代码：</p>
<figure class="highlight java"><table><tbody><tr><td class="code"><pre><span class="line"><span class="comment">// SelectionKey.OP_READ 或 SelectionKey.OP_ACCEPT 就绪</span></span><br><span class="line"><span class="comment">// readyOps == 0 是对 JDK Bug 的处理，防止空的死循环</span></span><br><span class="line"><span class="comment">// Also check for readOps of 0 to workaround possible JDK bug which may otherwise lead</span></span><br><span class="line"><span class="comment">// to a spin loop</span></span><br><span class="line"><span class="keyword">if</span> ((readyOps &amp; (SelectionKey.OP_READ | SelectionKey.OP_ACCEPT)) != <span class="number">0</span> || readyOps == <span class="number">0</span>) {</span><br><span class="line">    unsafe.read();</span><br><span class="line">}</span><br></pre></td></tr></tbody></table></figure>
<ul>
<li>当 <code>(readyOps &amp; SelectionKey.OP_READ) != 0</code> 时，这就是 NioSocketChannel 所在的 EventLoop 的线程<strong>轮询到</strong>有新的数据写入。</li>
<li>然后，调用 <code>NioByteUnsafe#read()</code> 方法，读取新的写入数据。</li>
</ul>
<hr>
<p><code>NioByteUnsafe#read()</code> 方法，读取新的写入数据。代码如下：</p>
<figure class="highlight java"><table><tbody><tr><td class="code"><pre><span class="line"> <span class="number">1</span>: <span class="meta">@Override</span></span><br><span class="line"> <span class="number">2</span>: <span class="meta">@SuppressWarnings</span>(<span class="string">"Duplicates"</span>)</span><br><span class="line"> <span class="number">3</span>: <span class="function"><span class="keyword">public</span> <span class="keyword">final</span> <span class="keyword">void</span> <span class="title">read</span><span class="params">()</span> </span>{</span><br><span class="line"> <span class="number">4</span>:     <span class="keyword">final</span> ChannelConfig config = config();</span><br><span class="line"> <span class="number">5</span>:     <span class="comment">// 若 inputClosedSeenErrorOnRead = true ，移除对 SelectionKey.OP_READ 事件的感兴趣。</span></span><br><span class="line"> <span class="number">6</span>:     <span class="keyword">if</span> (shouldBreakReadReady(config)) {</span><br><span class="line"> <span class="number">7</span>:         clearReadPending();</span><br><span class="line"> <span class="number">8</span>:         <span class="keyword">return</span>;</span><br><span class="line"> <span class="number">9</span>:     }</span><br><span class="line"><span class="number">10</span>:     <span class="keyword">final</span> ChannelPipeline pipeline = pipeline();</span><br><span class="line"><span class="number">11</span>:     <span class="keyword">final</span> ByteBufAllocator allocator = config.getAllocator();</span><br><span class="line"><span class="number">12</span>:     <span class="comment">// 获得 RecvByteBufAllocator.Handle 对象</span></span><br><span class="line"><span class="number">13</span>:     <span class="keyword">final</span> RecvByteBufAllocator.Handle allocHandle = recvBufAllocHandle();</span><br><span class="line"><span class="number">14</span>:     <span class="comment">// 重置 RecvByteBufAllocator.Handle 对象</span></span><br><span class="line"><span class="number">15</span>:     allocHandle.reset(config);</span><br><span class="line"><span class="number">16</span>: </span><br><span class="line"><span class="number">17</span>:     ByteBuf byteBuf = <span class="keyword">null</span>;</span><br><span class="line"><span class="number">18</span>:     <span class="keyword">boolean</span> close = <span class="keyword">false</span>; <span class="comment">// 是否关闭连接</span></span><br><span class="line"><span class="number">19</span>:     <span class="keyword">try</span> {</span><br><span class="line"><span class="number">20</span>:         <span class="keyword">do</span> {</span><br><span class="line"><span class="number">21</span>:             <span class="comment">// 申请 ByteBuf 对象</span></span><br><span class="line"><span class="number">22</span>:             byteBuf = allocHandle.allocate(allocator);</span><br><span class="line"><span class="number">23</span>:             <span class="comment">// 读取数据</span></span><br><span class="line"><span class="number">24</span>:             <span class="comment">// 设置最后读取字节数</span></span><br><span class="line"><span class="number">25</span>:             allocHandle.lastBytesRead(doReadBytes(byteBuf));</span><br><span class="line"><span class="number">26</span>:             <span class="comment">// &lt;1&gt; 未读取到数据</span></span><br><span class="line"><span class="number">27</span>:             <span class="keyword">if</span> (allocHandle.lastBytesRead() &lt;= <span class="number">0</span>) {</span><br><span class="line"><span class="number">28</span>:                 <span class="comment">// 释放 ByteBuf 对象</span></span><br><span class="line"><span class="number">29</span>:                 <span class="comment">// nothing was read. release the buffer.</span></span><br><span class="line"><span class="number">30</span>:                 byteBuf.release();</span><br><span class="line"><span class="number">31</span>:                 <span class="comment">// 置空 ByteBuf 对象</span></span><br><span class="line"><span class="number">32</span>:                 byteBuf = <span class="keyword">null</span>;</span><br><span class="line"><span class="number">33</span>:                 <span class="comment">// 如果最后读取的字节为小于 0 ，说明对端已经关闭</span></span><br><span class="line"><span class="number">34</span>:                 close = allocHandle.lastBytesRead() &lt; <span class="number">0</span>;</span><br><span class="line"><span class="number">35</span>:                 <span class="comment">// TODO</span></span><br><span class="line"><span class="number">36</span>:                 <span class="keyword">if</span> (close) {</span><br><span class="line"><span class="number">37</span>:                     <span class="comment">// There is nothing left to read as we received an EOF.</span></span><br><span class="line"><span class="number">38</span>:                     readPending = <span class="keyword">false</span>;</span><br><span class="line"><span class="number">39</span>:                 }</span><br><span class="line"><span class="number">40</span>:                 <span class="comment">// 结束循环</span></span><br><span class="line"><span class="number">41</span>:                 <span class="keyword">break</span>;</span><br><span class="line"><span class="number">42</span>:             }</span><br><span class="line"><span class="number">43</span>: </span><br><span class="line"><span class="number">44</span>:             <span class="comment">// &lt;2&gt; 读取到数据</span></span><br><span class="line"><span class="number">45</span>: </span><br><span class="line"><span class="number">46</span>:             <span class="comment">// 读取消息数量 + localRead</span></span><br><span class="line"><span class="number">47</span>:             allocHandle.incMessagesRead(<span class="number">1</span>);</span><br><span class="line"><span class="number">48</span>:             <span class="comment">// TODO 芋艿 readPending</span></span><br><span class="line"><span class="number">49</span>:             readPending = <span class="keyword">false</span>;</span><br><span class="line"><span class="number">50</span>:             <span class="comment">// 触发 Channel read 事件到 pipeline 中。 TODO</span></span><br><span class="line"><span class="number">51</span>:             pipeline.fireChannelRead(byteBuf);</span><br><span class="line"><span class="number">52</span>:             <span class="comment">// 置空 ByteBuf 对象</span></span><br><span class="line"><span class="number">53</span>:             byteBuf = <span class="keyword">null</span>;</span><br><span class="line"><span class="number">54</span>:         } <span class="keyword">while</span> (allocHandle.continueReading()); <span class="comment">// 循环判断是否继续读取</span></span><br><span class="line"><span class="number">55</span>: </span><br><span class="line"><span class="number">56</span>:         <span class="comment">// 读取完成</span></span><br><span class="line"><span class="number">57</span>:         allocHandle.readComplete();</span><br><span class="line"><span class="number">58</span>:         <span class="comment">// 触发 Channel readComplete 事件到 pipeline 中。</span></span><br><span class="line"><span class="number">59</span>:         pipeline.fireChannelReadComplete();</span><br><span class="line"><span class="number">60</span>: </span><br><span class="line"><span class="number">61</span>:         <span class="comment">// 关闭客户端的连接</span></span><br><span class="line"><span class="number">62</span>:         <span class="keyword">if</span> (close) {</span><br><span class="line"><span class="number">63</span>:             closeOnRead(pipeline);</span><br><span class="line"><span class="number">64</span>:         }</span><br><span class="line"><span class="number">65</span>:     } <span class="keyword">catch</span> (Throwable t) {</span><br><span class="line"><span class="number">66</span>:         handleReadException(pipeline, byteBuf, t, close, allocHandle);</span><br><span class="line"><span class="number">67</span>:     } <span class="keyword">finally</span> {</span><br><span class="line"><span class="number">68</span>:         <span class="comment">// TODO 芋艿 readPending</span></span><br><span class="line"><span class="number">69</span>:         <span class="comment">// Check if there is a readPending which was not processed yet.</span></span><br><span class="line"><span class="number">70</span>:         <span class="comment">// This could be for two reasons:</span></span><br><span class="line"><span class="number">71</span>:         <span class="comment">// * The user called Channel.read() or ChannelHandlerContext.read() in channelRead(...) method</span></span><br><span class="line"><span class="number">72</span>:         <span class="comment">// * The user called Channel.read() or ChannelHandlerContext.read() in channelReadComplete(...) method</span></span><br><span class="line"><span class="number">73</span>:         <span class="comment">//</span></span><br><span class="line"><span class="number">74</span>:         <span class="comment">// See https://github.com/netty/netty/issues/2254</span></span><br><span class="line"><span class="number">75</span>:         <span class="keyword">if</span> (!readPending &amp;&amp; !config.isAutoRead()) {</span><br><span class="line"><span class="number">76</span>:             removeReadOp();</span><br><span class="line"><span class="number">77</span>:         }</span><br><span class="line"><span class="number">78</span>:     }</span><br><span class="line"><span class="number">79</span>: }</span><br></pre></td></tr></tbody></table></figure>
<ul>
<li>第 5 至 9 行：若 inputClosedSeenErrorOnRead = true ，移除对 SelectionKey.OP_READ 事件的感兴趣。详细解析，见 <a href="http://svip.iocoder.cn/Netty/Channel-7-close/">《精尽 Netty 源码解析 —— Channel（七）之 close 操作》</a> 的 <a href="#">「5. 服务端处理客户端主动关闭连接」</a> 小节。</li>
<li>第 12 至 15 行：获得 RecvByteBufAllocator.Handle 对象，并重置它。这里的逻辑，和 <code>NioMessageUnsafe#read()</code> 方法的【第 14 至 17 行】的代码是一致的。相关的解析，见 <a href="http://svip.iocoder.cn/Netty/Channel-2-accept">《精尽 Netty 源码解析 —— Channel（二）之 accept 操作》</a> 。</li>
<li><p>第 20 至 64 行：<strong>while 循环</strong> 读取新的写入数据。</p>
<ul>
<li>第 22 行：调用 <code>RecvByteBufAllocator.Handle#allocate(ByteBufAllocator allocator)</code> 方法，申请 ByteBuf 对象。关于它的内容，我们放在 ByteBuf 相关的文章，详细解析。</li>
<li>第 25 行：调用 <code>AbstractNioByteChannel#doReadBytes(ByteBuf buf)</code> 方法，读取数据。详细解析，胖友先跳到 <a href="#">「3. AbstractNioMessageChannel#doReadMessages」</a> 中，看完记得回到此处。</li>
<li><p>第 25 行：调用 <code>RecvByteBufAllocator.Handle#lastBytesRead(int bytes)</code> 方法，设置<strong>最后</strong>读取字节数。代码如下：</p>
<figure class="highlight java"><table><tbody><tr><td class="code"><pre><span class="line"><span class="comment">// AdaptiveRecvByteBufAllocator.HandleImpl.java</span></span><br><span class="line"><span class="meta">@Override</span></span><br><span class="line"><span class="function"><span class="keyword">public</span> <span class="keyword">void</span> <span class="title">lastBytesRead</span><span class="params">(<span class="keyword">int</span> bytes)</span> </span>{</span><br><span class="line">    <span class="comment">// If we read as much as we asked for we should check if we need to ramp up the size of our next guess.</span></span><br><span class="line">    <span class="comment">// This helps adjust more quickly when large amounts of data is pending and can avoid going back to</span></span><br><span class="line">    <span class="comment">// the selector to check for more data. Going back to the selector can add significant latency for large</span></span><br><span class="line">    <span class="comment">// data transfers.</span></span><br><span class="line">    <span class="keyword">if</span> (bytes == attemptedBytesRead()) {</span><br><span class="line">        record(bytes);</span><br><span class="line">    }</span><br><span class="line">    <span class="keyword">super</span>.lastBytesRead(bytes);</span><br><span class="line">}</span><br><span class="line"></span><br><span class="line"><span class="comment">// DefaultMaxMessagesRecvByteBufAllocator.MaxMessageHandle.java</span></span><br><span class="line"><span class="meta">@Override</span></span><br><span class="line"><span class="function"><span class="keyword">public</span> <span class="keyword">void</span> <span class="title">lastBytesRead</span><span class="params">(<span class="keyword">int</span> bytes)</span> </span>{</span><br><span class="line">    lastBytesRead = bytes; <span class="comment">// 设置最后一次读取字节数 &lt;1&gt;</span></span><br><span class="line">    <span class="keyword">if</span> (bytes &gt; <span class="number">0</span>) {</span><br><span class="line">        totalBytesRead += bytes; <span class="comment">// 总共读取字节数</span></span><br><span class="line">    }</span><br><span class="line">}</span><br></pre></td></tr></tbody></table></figure>
<ul>
<li>代码比较多，我们只看重点，当然也不细讲。</li>
<li>在 <code>&lt;1&gt;</code> 处，设置最后一次读取字节数。</li>
</ul>
</li>
<li><p>读取有，有两种结果，<strong>是</strong>/<strong>否</strong>读取到数据。</p>
</li>
<li><code>&lt;1&gt;</code> <strong>未</strong>读取到数据，即 <code>allocHandle.lastBytesRead() &lt;= 0</code> 。</li>
<li>第 30 行：调用 <code>ByteBuf#release()</code> 方法，释放 ByteBuf 对象。<ul>
<li>第 32 行：置空 ByteBuf 对象。</li>
</ul>
</li>
<li>第 34 行：如果最后读取的字节为小于 0 ，说明对端已经关闭。</li>
<li>第 35 至 39 行：TODO 芋艿 细节</li>
<li>第 41 行：<code>break</code> 结束循环。</li>
<li><code>&lt;2&gt;</code> <strong>有</strong>读取到数据，即 <code>allocHandle.lastBytesRead() &gt; 0</code> 。</li>
<li>第 47 行：调用 <code>AdaptiveRecvByteBufAllocator.HandleImpl#incMessagesRead(int amt)</code> 方法，读取消息( 客户端 )数量 + <code>localRead = 1</code> 。</li>
<li>第 49 行：TODO 芋艿 readPending</li>
<li><p>第 51 行：调用 <code>ChannelPipeline#fireChannelRead(Object msg)</code> 方法，触发 Channel read 事件到 pipeline 中。</p>
<ul>
<li><strong>注意</strong>，一般情况下，我们会在自己的 Netty 应用程序中，自定义 ChannelHandler 处理读取到的数据。😈 当然，此时读取的数据，大多数情况下是需要在解码( Decode )。关于这一块，在后续关于 Codec ( 编解码 )的文章中，详细解析。</li>
<li><p>如果没有自定义 ChannelHandler 进行处理，最终会被 pipeline 中的尾节点  TailContext 所处理。代码如下：</p>
<figure class="highlight java"><table><tbody><tr><td class="code"><pre><span class="line"><span class="comment">// TailContext.java</span></span><br><span class="line"><span class="meta">@Override</span></span><br><span class="line"><span class="function"><span class="keyword">public</span> <span class="keyword">void</span> <span class="title">channelRead</span><span class="params">(ChannelHandlerContext ctx, Object msg)</span> <span class="keyword">throws</span> Exception </span>{</span><br><span class="line">    onUnhandledInboundMessage(msg);</span><br><span class="line">}</span><br><span class="line"></span><br><span class="line"><span class="comment">// DefaultChannelPipeline.java</span></span><br><span class="line"><span class="function"><span class="keyword">protected</span> <span class="keyword">void</span> <span class="title">onUnhandledInboundMessage</span><span class="params">(Object msg)</span> </span>{</span><br><span class="line">    <span class="keyword">try</span> {</span><br><span class="line">        logger.debug(<span class="string">"Discarded inbound message {} that reached at the tail of the pipeline. "</span> + <span class="string">"Please check your pipeline configuration."</span>, msg);</span><br><span class="line">    } <span class="keyword">finally</span> {</span><br><span class="line">        ReferenceCountUtil.release(msg);</span><br><span class="line">    }</span><br><span class="line">}</span><br></pre></td></tr></tbody></table></figure>
<ul>
<li>最终也会<strong>释放</strong> ByteBuf 对象。这就是为什么【第 53 行】的代码，只去置空 ByteBuf 对象，而不用再去释放的原因。</li>
</ul>
</li>
</ul>
</li>
<li>第 53 行：置空 ByteBuf 对象。</li>
<li><p>第 54 行：调用 <code>AdaptiveRecvByteBufAllocator.HandleImpl#incMessagesRead(int amt)#continueReading()</code> 方法，判断是否循环是否继续，读取新的数据。代码如下： </p>
<figure class="highlight java"><table><tbody><tr><td class="code"><pre><span class="line"><span class="comment">// DefaultMaxMessagesRecvByteBufAllocator.MaxMessageHandle.java</span></span><br><span class="line"><span class="keyword">private</span> <span class="keyword">final</span> UncheckedBooleanSupplier defaultMaybeMoreSupplier = <span class="keyword">new</span> UncheckedBooleanSupplier() {</span><br><span class="line">    <span class="meta">@Override</span></span><br><span class="line">    <span class="function"><span class="keyword">public</span> <span class="keyword">boolean</span> <span class="title">get</span><span class="params">()</span> </span>{</span><br><span class="line">        <span class="keyword">return</span> attemptedBytesRead == lastBytesRead; <span class="comment">// 最后读取的字节数，是否等于，最大可写入的字节数</span></span><br><span class="line">    }</span><br><span class="line">};</span><br><span class="line"></span><br><span class="line"><span class="meta">@Override</span></span><br><span class="line"><span class="function"><span class="keyword">public</span> <span class="keyword">boolean</span> <span class="title">continueReading</span><span class="params">()</span> </span>{</span><br><span class="line">    <span class="keyword">return</span> continueReading(defaultMaybeMoreSupplier);</span><br><span class="line">}</span><br><span class="line"></span><br><span class="line"><span class="meta">@Override</span></span><br><span class="line"><span class="function"><span class="keyword">public</span> <span class="keyword">boolean</span> <span class="title">continueReading</span><span class="params">(UncheckedBooleanSupplier maybeMoreDataSupplier)</span> </span>{</span><br><span class="line">    <span class="keyword">return</span> config.isAutoRead() &amp;&amp;</span><br><span class="line">           (!respectMaybeMoreData || maybeMoreDataSupplier.get()) &amp;&amp; <span class="comment">// &lt;1&gt;</span></span><br><span class="line">           totalMessages &lt; maxMessagePerRead &amp;&amp;</span><br><span class="line">           totalBytesRead &gt; <span class="number">0</span>;</span><br><span class="line">}</span><br></pre></td></tr></tbody></table></figure>
<ul>
<li>一般情况下，最后读取的字节数，<strong>不等于</strong>最大可写入的字节数，即 <code>&lt;1&gt;</code> 处的代码 <code>UncheckedBooleanSupplier#get()</code> 返回 <code>false</code> ，则不再进行数据读取。因为 😈 也没有数据可以读取啦。</li>
</ul>
</li>
</ul>
</li>
<li>第 57 行：调用 <code>RecvByteBufAllocator.Handle#readComplete()</code> 方法，读取完成。暂无重要的逻辑，不详细解析。</li>
<li><p>第 59 行：调用 <code>ChannelPipeline#fireChannelReadComplete()</code> 方法，触发 Channel readComplete 事件到 pipeline 中。</p>
<ul>
<li><em>如果有需要，胖友可以自定义处理器，处理该事件。一般情况下，不需要</em>。</li>
<li><p>如果没有自定义 ChannelHandler 进行处理，最终会被 pipeline 中的尾节点  TailContext 所处理。代码如下：</p>
<figure class="highlight java"><table><tbody><tr><td class="code"><pre><span class="line"><span class="comment">// TailContext.java</span></span><br><span class="line"><span class="meta">@Override</span></span><br><span class="line"><span class="function"><span class="keyword">public</span> <span class="keyword">void</span> <span class="title">channelReadComplete</span><span class="params">(ChannelHandlerContext ctx)</span> <span class="keyword">throws</span> Exception </span>{</span><br><span class="line">    onUnhandledInboundChannelReadComplete();</span><br><span class="line">}</span><br><span class="line"></span><br><span class="line"><span class="comment">// DefaultChannelPipeline.java</span></span><br><span class="line"><span class="function"><span class="keyword">protected</span> <span class="keyword">void</span> <span class="title">onUnhandledInboundChannelReadComplete</span><span class="params">()</span> </span>{</span><br><span class="line">}</span><br></pre></td></tr></tbody></table></figure>
<ul>
<li>具体的调用是<strong>空方法</strong>。</li>
</ul>
</li>
</ul>
</li>
<li>第 61 至 64 行：关闭客户端的连接。详细解析，见 <a href="http://svip.iocoder.cn/Netty/Channel-7-close/">《精尽 Netty 源码解析 —— Channel（七）之 close 操作》</a> 的 <a href="#">「5. 服务端处理客户端主动关闭连接」</a> 小节。</li>
<li>第 65 至 66 行：当发生异常时，调用 <code>#handleReadException(hannelPipeline pipeline, ByteBuf byteBuf, Throwable cause, boolean close, RecvByteBufAllocator.Handle allocHandle)</code> 方法，处理异常。详细解析，见 <a href="#">「2.2 handleReadException」</a> 中。</li>
<li>第 67 至 78 行：TODO 芋艿 细节</li>
</ul>
<h2 id="2-2-handleReadException"><a href="#2-2-handleReadException" class="headerlink" title="2.2 handleReadException"></a>2.2 handleReadException</h2><p><code>#handleReadException(hannelPipeline pipeline, ByteBuf byteBuf, Throwable cause, boolean close, RecvByteBufAllocator.Handle allocHandle)</code> 方法，处理异常。代码如下：</p>
<figure class="highlight java"><table><tbody><tr><td class="code"><pre><span class="line"> <span class="number">1</span>: <span class="function"><span class="keyword">private</span> <span class="keyword">void</span> <span class="title">handleReadException</span><span class="params">(ChannelPipeline pipeline, ByteBuf byteBuf, Throwable cause, <span class="keyword">boolean</span> close, RecvByteBufAllocator.Handle allocHandle)</span> </span>{</span><br><span class="line"> <span class="number">2</span>:     <span class="keyword">if</span> (byteBuf != <span class="keyword">null</span>) {</span><br><span class="line"> <span class="number">3</span>:         <span class="keyword">if</span> (byteBuf.isReadable()) {</span><br><span class="line"> <span class="number">4</span>:             <span class="comment">// TODO 芋艿 细节</span></span><br><span class="line"> <span class="number">5</span>:             readPending = <span class="keyword">false</span>;</span><br><span class="line"> <span class="number">6</span>:             <span class="comment">// 触发 Channel read 事件到 pipeline 中。</span></span><br><span class="line"> <span class="number">7</span>:             pipeline.fireChannelRead(byteBuf);</span><br><span class="line"> <span class="number">8</span>:         } <span class="keyword">else</span> {</span><br><span class="line"> <span class="number">9</span>:             <span class="comment">// 释放 ByteBuf 对象</span></span><br><span class="line"><span class="number">10</span>:             byteBuf.release();</span><br><span class="line"><span class="number">11</span>:         }</span><br><span class="line"><span class="number">12</span>:     }</span><br><span class="line"><span class="number">13</span>:     <span class="comment">// 读取完成</span></span><br><span class="line"><span class="number">14</span>:     allocHandle.readComplete();</span><br><span class="line"><span class="number">15</span>:     <span class="comment">// 触发 Channel readComplete 事件到 pipeline 中。</span></span><br><span class="line"><span class="number">16</span>:     pipeline.fireChannelReadComplete();</span><br><span class="line"><span class="number">17</span>:     <span class="comment">// 触发 exceptionCaught 事件到 pipeline 中。</span></span><br><span class="line"><span class="number">18</span>:     pipeline.fireExceptionCaught(cause);</span><br><span class="line"><span class="number">19</span>:     <span class="comment">// // TODO 芋艿 细节</span></span><br><span class="line"><span class="number">20</span>:     <span class="keyword">if</span> (close || cause <span class="keyword">instanceof</span> IOException) {</span><br><span class="line"><span class="number">21</span>:         closeOnRead(pipeline);</span><br><span class="line"><span class="number">22</span>:     }</span><br><span class="line"><span class="number">23</span>: }</span><br></pre></td></tr></tbody></table></figure>
<ul>
<li><p>第 2 行：<code>byteBuf</code> 非空，说明在发生异常之前，至少申请 ByteBuf 对象是<strong>成功</strong>的。</p>
<ul>
<li><p>第 3 行：调用 <code>ByteBuf#isReadable()</code> 方法，判断 ByteBuf 对象是否可读，即剩余可读的字节数据。</p>
<ul>
<li><p>该方法的英文注释如下：</p>
<figure class="highlight java"><table><tbody><tr><td class="code"><pre><span class="line"><span class="comment">/**</span></span><br><span class="line"><span class="comment"> * Returns {<span class="doctag">@code</span> true}</span></span><br><span class="line"><span class="comment"> * if and only if {<span class="doctag">@code</span> (this.writerIndex - this.readerIndex)} is greater</span></span><br><span class="line"><span class="comment"> * than {<span class="doctag">@code</span> 0}.</span></span><br><span class="line"><span class="comment"> */</span></span><br><span class="line"><span class="function"><span class="keyword">public</span> <span class="keyword">abstract</span> <span class="keyword">boolean</span> <span class="title">isReadable</span><span class="params">()</span></span>;</span><br></pre></td></tr></tbody></table></figure>
<ul>
<li>即 <code>this.writerIndex - this.readerIndex &gt; 0</code> 。</li>
</ul>
</li>
<li>第 5 行：TODO 芋艿 细节</li>
<li>第 7 行：调用 <code>ChannelPipeline#fireChannelRead(Object msg)</code> 方法，触发 Channel read 事件到 pipeline 中。</li>
</ul>
</li>
<li>第 8 至 11 行：ByteBuf 对象不可读，所以调用 <code>ByteBuf#release()</code> 方法，释放 ByteBuf 对象。</li>
</ul>
</li>
<li>第 14 行：调用 <code>RecvByteBufAllocator.Handle#readComplete()</code> 方法，读取完成。暂无重要的逻辑，不详细解析。</li>
<li>第 16 行：调用 <code>ChannelPipeline#fireChannelReadComplete()</code> 方法，触发 Channel readComplete 事件到 pipeline 中。</li>
<li><p>第 18 行：调用 <code>ChannelPipeline#fireExceptionCaught(Throwable)</code> 方法，触发 exceptionCaught 事件到 pipeline 中。</p>
<ul>
<li><strong>注意</strong>，一般情况下，我们会在自己的 Netty 应用程序中，自定义 ChannelHandler 处理异常。</li>
<li><p>如果没有自定义 ChannelHandler 进行处理，最终会被 pipeline 中的尾节点  TailContext 所处理。代码如下：</p>
<figure class="highlight java"><table><tbody><tr><td class="code"><pre><span class="line"><span class="comment">// TailContext.java</span></span><br><span class="line"><span class="meta">@Override</span></span><br><span class="line"><span class="function"><span class="keyword">public</span> <span class="keyword">void</span> <span class="title">exceptionCaught</span><span class="params">(ChannelHandlerContext ctx, Throwable cause)</span> <span class="keyword">throws</span> Exception </span>{</span><br><span class="line">    onUnhandledInboundException(cause);</span><br><span class="line">}</span><br><span class="line"></span><br><span class="line"><span class="comment">// DefaultChannelPipeline.java</span></span><br><span class="line"><span class="function"><span class="keyword">protected</span> <span class="keyword">void</span> <span class="title">onUnhandledInboundException</span><span class="params">(Throwable cause)</span> </span>{</span><br><span class="line">    <span class="keyword">try</span> {</span><br><span class="line">        logger.warn(<span class="string">"An exceptionCaught() event was fired, and it reached at the tail of the pipeline. "</span> +</span><br><span class="line">                        <span class="string">"It usually means the last handler in the pipeline did not handle the exception."</span>,</span><br><span class="line">                cause);</span><br><span class="line">    } <span class="keyword">finally</span> {</span><br><span class="line">        ReferenceCountUtil.release(cause);</span><br><span class="line">    }</span><br><span class="line">}</span><br></pre></td></tr></tbody></table></figure>
<ul>
<li>打印<strong>告警</strong>日志。</li>
<li>调用 <code>ReferenceCountUtil#release(Object msg)</code> 方法，释放和异常相关的资源。</li>
</ul>
</li>
</ul>
</li>
<li>第 19 至 22 行：TODO 芋艿，细节</li>
</ul>
<h2 id="2-3-closeOnRead"><a href="#2-3-closeOnRead" class="headerlink" title="2.3 closeOnRead"></a>2.3 closeOnRead</h2><p>TODO 芋艿，细节</p>
<h1 id="3-AbstractNioByteChannel-doReadBytes"><a href="#3-AbstractNioByteChannel-doReadBytes" class="headerlink" title="3. AbstractNioByteChannel#doReadBytes"></a>3. AbstractNioByteChannel#doReadBytes</h1><p><code>doReadBytes(ByteBuf buf)</code> <strong>抽象</strong>方法，读取写入的数据到方法参数 <code>buf</code> 中。它是一个<strong>抽象</strong>方法，定义在 AbstractNioByteChannel 抽象类中。代码如下：</p>
<figure class="highlight java"><table><tbody><tr><td class="code"><pre><span class="line"><span class="comment">/**</span></span><br><span class="line"><span class="comment"> * Read bytes into the given {<span class="doctag">@link</span> ByteBuf} and return the amount.</span></span><br><span class="line"><span class="comment"> */</span></span><br><span class="line"><span class="function"><span class="keyword">protected</span> <span class="keyword">abstract</span> <span class="keyword">int</span> <span class="title">doReadBytes</span><span class="params">(ByteBuf buf)</span> <span class="keyword">throws</span> Exception</span>;</span><br></pre></td></tr></tbody></table></figure>
<ul>
<li>返回值为读取到的字节数。</li>
<li><strong>当返回值小于 0 时，表示对端已经关闭</strong>。</li>
</ul>
<p>NioSocketChannel 对该方法的实现代码如下：</p>
<figure class="highlight java"><table><tbody><tr><td class="code"><pre><span class="line"><span class="number">1</span>: <span class="meta">@Override</span></span><br><span class="line"><span class="number">2</span>: <span class="function"><span class="keyword">protected</span> <span class="keyword">int</span> <span class="title">doReadBytes</span><span class="params">(ByteBuf byteBuf)</span> <span class="keyword">throws</span> Exception </span>{</span><br><span class="line"><span class="number">3</span>:     <span class="comment">// 获得 RecvByteBufAllocator.Handle 对象</span></span><br><span class="line"><span class="number">4</span>:     <span class="keyword">final</span> RecvByteBufAllocator.Handle allocHandle = unsafe().recvBufAllocHandle();</span><br><span class="line"><span class="number">5</span>:     <span class="comment">// 设置最大可读取字节数量。因为 ByteBuf 目前最大写入的大小为 byteBuf.writableBytes()</span></span><br><span class="line"><span class="number">6</span>:     allocHandle.attemptedBytesRead(byteBuf.writableBytes());</span><br><span class="line"><span class="number">7</span>:     <span class="comment">// 读取数据到 ByteBuf 中</span></span><br><span class="line"><span class="number">8</span>:     <span class="keyword">return</span> byteBuf.writeBytes(javaChannel(), allocHandle.attemptedBytesRead());</span><br><span class="line"><span class="number">9</span>: }</span><br></pre></td></tr></tbody></table></figure>
<ul>
<li>第 4 行：获得 RecvByteBufAllocator.Handle 对象。<ul>
<li>第 6 行：设置最大可读取字节数量。因为 ByteBuf 对象<strong>目前</strong>最大可写入的大小为 <code>ByteBuf#writableBytes()</code> 的长度。</li>
</ul>
</li>
<li><p>第 8 行：调用 <code>ByteBuf#writeBytes(ScatteringByteChannel in, int length)</code> 方法，读取数据到 ByteBuf 对象中。因为 ByteBuf 有多种实现，我们以默认的 PooledUnsafeDirectByteBuf 举例子。代码如下：</p>
<figure class="highlight java"><table><tbody><tr><td class="code"><pre><span class="line"><span class="comment">// AbstractByteBuf.java</span></span><br><span class="line"><span class="meta">@Override</span></span><br><span class="line"><span class="function"><span class="keyword">public</span> <span class="keyword">int</span> <span class="title">writeBytes</span><span class="params">(ScatteringByteChannel in, <span class="keyword">int</span> length)</span> <span class="keyword">throws</span> IOException </span>{</span><br><span class="line">    ensureWritable(length);</span><br><span class="line">    <span class="keyword">int</span> writtenBytes = setBytes(writerIndex, in, length); <span class="comment">// &lt;1&gt;</span></span><br><span class="line">    <span class="keyword">if</span> (writtenBytes &gt; <span class="number">0</span>) { <span class="comment">// &lt;3&gt;</span></span><br><span class="line">        writerIndex += writtenBytes;</span><br><span class="line">    }</span><br><span class="line">    <span class="keyword">return</span> writtenBytes;</span><br><span class="line">}</span><br><span class="line"></span><br><span class="line"><span class="comment">// PooledUnsafeDirectByteBuf.java</span></span><br><span class="line"><span class="meta">@Override</span></span><br><span class="line"><span class="function"><span class="keyword">public</span> <span class="keyword">int</span> <span class="title">setBytes</span><span class="params">(<span class="keyword">int</span> index, ScatteringByteChannel in, <span class="keyword">int</span> length)</span> <span class="keyword">throws</span> IOException </span>{</span><br><span class="line">    checkIndex(index, length);</span><br><span class="line">    ByteBuffer tmpBuf = internalNioBuffer();</span><br><span class="line">    index = idx(index);</span><br><span class="line">    tmpBuf.clear().position(index).limit(index + length);</span><br><span class="line">    <span class="keyword">try</span> {</span><br><span class="line">        <span class="keyword">return</span> in.read(tmpBuf); <span class="comment">// &lt;2&gt;</span></span><br><span class="line">    } <span class="keyword">catch</span> (ClosedChannelException ignored) {</span><br><span class="line">        <span class="keyword">return</span> -<span class="number">1</span>;</span><br><span class="line">    }</span><br><span class="line">}</span><br></pre></td></tr></tbody></table></figure>
<ul>
<li>代码比较多，我们只看重点，当然也不细讲。还是那句话，关于 ByteBuf 的内容，我们在 ByteBuf 相关的文章详细解析。</li>
<li>在 <code>&lt;1&gt;</code> 处，会调用 <code>#setBytes(int index, ScatteringByteChannel in, int length)</code> 方法。</li>
<li>在 <code>&lt;2&gt;</code> 处，会调用 Java NIO 的 <code>ScatteringByteChannel#read(ByteBuffer)</code> 方法，读取<strong>数据</strong>到临时的 Java NIO ByteBuffer 中。<ul>
<li>在对端未断开时，返回的是读取数据的<strong>字节数</strong>。</li>
<li>在对端已断开时，返回 <code>-1</code> ，表示断开。这也是为什么 <code>&lt;3&gt;</code> 处做了 <code>writtenBytes &gt; 0</code> 的判断的原因。</li>
</ul>
</li>
</ul>
</li>
</ul>
<h1 id="666-彩蛋"><a href="#666-彩蛋" class="headerlink" title="666. 彩蛋"></a>666. 彩蛋</h1><p>推荐阅读文章：</p>
<ul>
<li>闪电侠 <a href="https://www.jianshu.com/p/6b48196b5043" rel="external nofollow noopener noreferrer" target="_blank">《深入浅出 Netty read》</a></li>
<li>Hypercube <a href="https://www.jianshu.com/p/9258af254e1d" rel="external nofollow noopener noreferrer" target="_blank">《自顶向下深入分析Netty（六）– Channel源码实现》</a></li>
</ul>


</div>
<!--
<footer class="article-footer">
<a data-url="http://svip.iocoder.cn/Netty/Channel-3-read/" data-id="ck4pl3fp500e2fgcfrk29d8af" class="article-share-link">分享</a>



</footer>
-->
</div>