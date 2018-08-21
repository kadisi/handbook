# informerfactory 机制

## informerFactory 机制

1. informerFactory 相当于工厂模式，它可以存放多个informer，每个informer 主要包含几个关键元素：
   1. indexer 一个cashe 缓存
   2. processor： Processor 存放了多个listener
   3. listWatcher ： list 和 watch 对应对象的数据
   4. crontroller:  controller 里主要包含： listwatch，deltaFifo的队列，还有一个process
2. 每个informer 都可以添加Event，AddEvent操作，AddEvent 的参数是一个EventHandler，里面包含了AddFunction， UpdateFunction， DeleteFunction， 其最终的目的，是将informer ListAddWatch过来的数据 通过这三个Function，传递给外部对象
3. informer 每次add 一个Event时候，都会创建一个listener 然后放到Processor 里， listener 里面主要包含了几个关键要素：
   1. nextCh 和 addCh 的chan （无缓冲的chan） 以及 EventHandler，
   2. listener 创建后， 会启动listener 的run 和pop 功能， pop的实现：
      1. pop 的核心原理是将addCh chan里的数据转到nextCh channnel 中
   3. run 的实现， run是连接数据到Hander的转折点
      1. run 将nextCh 中的数据，取出来， 根据这个数据里的type 来判断是update，Add 还是Delete，然后执行handler 里的  AddFunction， UpdateFunction， DeleteFunction ，最终将数据转给了外部对象
   4. 那么 addCh 数据是怎么来的？ 最终又如何通过 AddFunction， UpdateFunction， DeleteFunction 转给了外部对象呢
   5. addCh 的由来：
      1. 最上层的InformerFactory 创建后，在通过添加各种Informer后（具体添加Informer的例子： s.InformerFactory.Core\(\).V1\(\).Nodes\(\) 这是添加node 的Informer）， 在最后会调用start 方法：
         1. start 核心逻辑是 调用里面每个Informer 的run 方法
      2. 每个Informer 的run 方法：可以说Informer的run 方法是一个生产者
         1. 在run 方法里，会创建一个DeltaFIFO 的队列，
         2. 会对controller 进行赋值， 将listwatch，deltafifo 赋值进去，另外是定义Process 方法。
            1. Process 方法的核心处理逻辑，是讲传给来的数据 1 更新informer 的indexr缓存。  2 是调用informer 的process 字段的distribute方法，distribute 方法的核心逻辑，是调用里面每个listener的add 方法，将数据放到 AddCh 的channel 中
         3. 之后Informer 会调用自己元素：process 的run方法，上面已经知道process 存放了很多listener， process 的run 方法是，通过go协程，执行每个listener 的run 和pop 方法，listener 的pop 和run 最终的目的是会将Informer得到的数据 传递给外部对象
         4. 接下来是最重要的control的run 方法，controller 是一个联系纽带，它的目的是通过一个机制，将listwatch 出来的数据，给每个listener 的 addCh 的channnel， 具体逻辑如下：
            1. 执行control 的run 方法后，会创建一个reflect的对象，reflect对象 主要保存了两个字段，一个是listwatch 另一个是 fifo 的队列，之后reflect 对象会执行run方法，核心逻辑就是调用apiserve的list and watch 方法，把数据放到fifo 的队列中
            2. 之后control 会执行 processLoop方法 这个loop方法 ，核心逻辑是不断的将fifo 队列中的数据pop出来，而pop出来的数据是给 control 的Process 字段代表的方法去处理。而Process 在上面也说了，正好是调用里面所有listener 的distribute 方法，将数据分发到每个listener 的addCh 字段中。
   6. 这样整个流程就跑通了，reflect 将apiserver 获得的数据放到fifo 队列中， controller的ProcessLoop方法，不断的将数据Pop 出来，给controller 的Process 去处理，Process 主要是讲获得的数据分发到informer的各个listener 的addCh channel 中，而每个listener都会执行pop 和run 操作，Pop 是讲addCh channel的数据转给nextCh 的channel 中， 而run 操作 是讲nextCh的数据取出来，最终调用AddFunction，或者UpdateFunction，或者DeleteFunction，之后就转给了外部对象，而这个三个function 是怎么转给外部对象的呢，一般情况下这外部对象一般也是一个cacher，如scheduler 会创建一个schedulercache， 而schedulercache会有对应的 AddFunction，或者UpdateFunction，或者DeleteFunction ，之后会将这些数据扔到新cache中
4. 5. 
