# 优化缓存性能的不同方法与心得
## 第一种：减少L1 Cache大小，降低Hit time，降低功耗
### 基本逻辑
<p> 1：首先明确，cache Hit的三个步骤：首先是用index查到tag，对比tag和tlb中查到的tag，如果是set associative缓存，还需要使用多路复用器对比。 显然，降低相关度将减少多路复用器的设计成本，减小功耗（PS：为了潜在读者，以及以后的自己，解释一下相关度是什么，其实就是4-way set associative 或者 8-way...里面这个数字）。</p>
<p>同理，其实cache里对比index值的电路也希望index的位数少一点，这样同样可以降低功耗，那么在cache size不变的情况下，可以增加cache block的大小。但是这样会导致cache miss增加。（书中没有解释原因，个人认为是index位数变少，tag位数变多，区分度下降。）以上是相关度，block需要控制的原因。</p>
<p>2：为什么现在的L1 cache还在提高相关度？我认为最关键的一点是：现在基本所有的L1 cache都是使用的virtual index，physical tag的方案。这样方案带来的问题是，cache可用的位数基本固定为16位，因为一个page的大小是4Kb，所以offset大小固定为16位。提高相关度可以加载更多的数据进入缓存，提高缓存大小。这个方案可以在虚拟地址翻译成物理地址前就对缓存进行索引。是个非常有吸引力的方案。还能为多线程带来的conflict Miss带来好处</p>

## 第二种：使用路预测缩短命中时间
### 基本逻辑
<p>这个很好理解，可以理解成在多路复用器那里加一个简单的分支预测器。一般来说，这种方案不一定能优化命中时间，更可能优化功耗。</p>

## 第三种：cache流水线化，或者使用multiBank技术
### 基本逻辑
<p> 流水线化的好处很明显，就是能够提高时钟频率，提高指令吞吐率。代价是增加单个指令的处理时延。  </p>
<p> 这里简单介绍一下什么是cache技术里说到的bank（自己看书理解的，不一定准确）。就是例如我们在逻辑上有一个大小为64Kb的L1 cache，我们现在想超标量地处理缓存命令，或者节约功耗。我们就可以在物理意义上，将64Kb的缓存，分成四个小块（bank），这样就可以同时读取多个不同板块中的数据，达到超标量的目的。同时如果想节约功耗，可以只处理一条缓存命令，那么在单位时间内，有且仅有一个bank被使用，因此也可以节约功耗。
值得注意的是，这样的技术希望我们平均地读取每个bank中的内容，如果内容集中在某个bank里，那么这个方法就失去了意义，所以在地址的分配上需要作出改动。例如0-15,这16个cache line。一个经典的分类方法是『0,4,8,12』，『1,5,9,13』，『2,6,10,14』，『3,7,11,15』，这样的方法叫做顺序交错（sequential interleaving），一号bank的行号是4的倍数，2号是模4余1,以此类推。这样能降低bank之间的冲突。  </p>

## 第四种：参用非阻塞缓存
### 基本逻辑
<p> 由于现在的CPU都是乱序执行的，因此处理器不必因为一次数据缓存缺失停顿。所以可以在缺失时，继续从指令缓存中取值。如果能重叠多个缺失，缓存就能进一步降低实际损失。但是这一方案的实现是非常麻烦的，会出现两种类型的挑战。首先是多个未解决的缺失之间会发生冲突，如何解决冲突，一般来说是为他们赋予优先级，或者在出现冲突时排序。 第二就是我们需要跟踪没有解决的缺失，在多个缺失冲突的背景下，返回的结果并不一定是按照顺序返回的，例如当我们一个L1中的miss在L2中可能也是miss也可能是hit，这两个不同的结果返回的时间显然不同。因此如何知道哪个缺失对应哪个结果呢。目前的处理器中使用了缺失状态处理寄存器（miss status handling register），如果支持N个缺失同时处理，则有N个MSHR。 </p>

## 第五种：合并写缓存区
### 基本逻辑
<p> 简单来说，就是写的时候，有个buffer在cpu和写缓存之间，cpu写在buffer里就当做完成了，这时我们应该等一下，合并多次写操作变成一个“大”写操作，如果有重复地址的写入，则直接更新该地址内容。 </p>

## 第六种：参用编译器优化
### 基本逻辑
<p> 这里主要讲解分块逻辑，分块逻辑本质上是借用了矩阵乘法的时间局限性。简单说来就是，把大矩阵乘法分解成小矩阵乘法。结果矩阵的每一行的前几列，需要用到列主序的矩阵的板块都是一样的。因此一次load的结果可以多次复用。详见（中文版85页） </p>

## 第七种：预取（硬件或者编译器）
### 基本逻辑
<p> 硬件预取一般在缓存外完成，例如取两个块，一个返回到缓存中，一个放在指令流buffer中，如果下次需要预取的数据，直接从buffer中送出。这样的方法需要空闲的存储带宽，有可能会干扰其他内容的访问。
另一种方法是，编译器优化，由编译器决定是否需要在特定位置插入预取指令，这带来两个问题，首先是增加了指令数量。所以需要确保开销小于好处。</p>
