# rs controller

1. 发现变化的rs 后，首先根据这个rs 的key 通过apiserver 获得这个rs 的对象
2. 获得这个rs 的label selector
3. 获得这个ns 下的所有pod
4. 从这些所有pod中过滤那些状态是删除的，faild，succed 的pod
5. 过滤完成后，开始索赔这些pod，索赔相当于过滤，同时会对这些pod 打patch，最后索赔出来的pod 代表了这个rs 下的pod列表，算法如下
   1. 首先看这个pod 的 ownerReferences 属性
      1. 如果有主了，
         1. 但是这个主并不是这个rs，抛弃掉
         2. 如果这个主是这个rs，而且pod 的标签满足rs 的label selector，要了，这个pod 选中合格
         3. 如果label selector 不满足，但是发现这个pod 是将要删除的，则抛弃掉
         4. 最后把这些有主，并且主是这个rs，并且标签不满足这个rs 的剩余pod release掉，就是去掉ownerReferences 的属性
      2. 如果没有主人
         1. 发现这个rs 本身就是将要删除的，或者pod 的label表情不满足这个rs，抛弃掉
         2. 发现这个pod 本身是将要删除的，抛弃掉
         3. 最后那些没有主人，pod 还不会删除，label 满足rs 的label selector 的，准备重新adopt，adopt 的做法就是给这个pod 的metadata 中加入ownerReferences属性， 代表 他属于哪个rs
6. 获得索赔的pod列表后，发现如果这个rs 需要同步，并且没有删除的标志，则开始执行manageReplicas逻辑，这是核心，算法如下
   1. 获得索赔出来的pod 数目，和rs 的replicas 做比较
   2. 如果少了，就扩容
   3. 如果多了，就删除，删除是有排序算法的
      1.  Unassigned &lt; assigned
      2.  PodPending &lt; PodUnknown &lt; PodRunning
      3.  Not ready &lt; ready
      4.  Been ready for empty time &lt; less time &lt; more time
      5.  Pods with containers with higher restart counts &lt; lower restart counts
      6.  Empty creation time pods &lt; newer pods &lt; older pods
7. 更新rs 的状态



