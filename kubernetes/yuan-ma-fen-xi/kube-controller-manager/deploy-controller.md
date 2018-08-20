# deploy controller



deploy controller 是一个不断修正的过程，这里主要讲RollingUpdate 算法，deploy 还有recreate 算法，这里就先不讲了



deploy controller 也是从informerfactory 获得更新或者创建的deploy 后，对deploy进行并发处理，在deploy controller 规定了同时可以并发处理的worker数目，默认是 30个，代表了同时能并发处理30个deploy



1. deploy 的正式逻辑如下， 注意 在RollingUpdate算法下，deploy controller 处理的完全是rs ，跟pod 没有任何鸟关系
   1. 根据从缓存中获得的deploy key，去从apiserver 中获得实时的这个deploy 信息，同时获得这个deploy 对应的所有rs列表，
   2. 如果发现这个deploy 是将要删除的（metadata中有个DeletionTimestamp字段不为空）， 则仅仅更新这个deploy 的状态，更新完后直接返回
      1. 主要是更新根据当前所有的rs 来更新deploy 的status：包括Replicas,UpdatedReplicas, UpdatedReplicas,UpdatedReplicas,UpdatedReplicas，UpdatedReplicas
   3. 接着下一逻辑， 检查deployment 的status 中的condition，如果当前deploy 是pause 的，则需要更新相应的condition状态，
   4. 接着下一逻辑，如果 这个deploy是pause 的（spec.Paused）, 则直接处理sync 逻辑，处理完后，直接返回，关于sync的逻辑如下（主要是对当前rs 状态下进行扩容和缩容）
      1. 获得这个deploy 对应的newrs 和所有oldrs， 执行scale 操作， scale 逻辑如下
         1. 从newrs 和oldrss 中找到最新或者最活跃的那一个（replicas 不为0），如果只有一个的话， 如果这个rs 的replicas 不等于deploy 的replicas， 则需要更新这个rs 的replicas，返回
         2. 如果有多个rs 的replicas 不为0 ， 并且newrs 的replicas= deploy 的replicas，则需要讲其他oldrss 的replicas 设置为0，返回
         3. 如果有多个rs 的replicas 不为0， 并且newrs 是不饱和的，即newrs 的replicas ！= deploy 的replicas。 这时候需要判断deploy 算法是否是 RollingUpdate， 如果是，则执行下面的逻辑：
            1. 这个逻辑完全是根据deploy的MaxSurge 和 和当前rs 下所有replicas 的总和，来计算 每个rs 下的最终replicas 值。计算完后，更新每个rs 的replicas值
            2. 注意，这个是不断微调的
      2. 执行完scalse，后，在执行更新deploy的status
   5. 接着下一个逻辑，如果发现这个deploy 的spec 有RollbackTo 值，代表了要rollback 到某个特定版本，则需要执行rollback 的逻辑，执行完后直接返回， rollback的逻辑如下：
      1. 找到这个deploy 的newRS 和所有oldrss
      2. 如果是回退到laster revision，处理相应的逻辑后，直接返回，逻辑如下
         1. 首先找到laster revison（倒数第二个大的revision）
         2. 遍历所有allrss， 找到revison 是laster revison 的rs，然后将这个rs 里的配置copy 给deploy 的template 中，更新这个deploy
      3. 
   6. 接着下一个逻辑，判断这个deploy 是否需要scale 
      1. 遍历所有的rs。 如果有一个rs 的spec.replicas 和 annotation中的desired 数目不一样，则需要执行sync操作
      2. 关于sync 操作，在第四部已经讲明
   7. 接着下一个逻辑，最后这个逻辑才是正常的rollupdate 的逻辑，你可以认为这一步是用户执行rollout 操作时候，才真正触发的， 执行完后直接返回，逻辑如下：
      1. 判断deploy 的策略是否是RollingUpdate，如果是rollingupdate， 则执行rolloutRolling 操作，逻辑如下
         1. 如果发现这个deploy 没有对应的rs，则直接创建这个rs，同事获得对应的旧的rs ：oldRSs
         2. 调和newRS,之后更新deploy 的状态
            1. 如果发现newRs 的replicas &gt; deploy 的replicas，要更新newRS 的replicas 为deploy 的replicas 然后返回
            2. 如果发现newrs 的replicas &lt; deploy 的replicas 需要更新newRs 的replicas，为一个特定的数目，这个数目是通过 NewRSNewReplicas 算法获得
         3.  调和所有oldRSs，之后更新deploy的状态
         4. 最后更新deploy 的状态
   8. 注意，deploy 的controller 的实际处理流程是一个不断自适应的过程，每个逻辑处理后，一般都会更新deploy，这样，informfactory 又回检测到这个deploy 的变化，但是这样会出现一个问题，如果deploy controller 的30个同时工作协程中，很可能会同时处理相同的deploy，很可能会出现死循环。那么如何避免这种情况呢，基本的解决思路就是，informerFactory 检测到deploy 发生变化后，会将变化的deploy 放到一个缓存中，只要这个缓存能够知道如果某个deploy正在处理时候，会禁止这个新更新的deploy继续重复操作就好了。
      1. 因此deploy controller 用了一个queue：workqueue.NewNamedRateLimitingQueue\(workqueue.DefaultControllerRateLimiter\(\), "deployment"\),
      2. 这个queue 支持 ADD，GET DELETE 操作。其实现是 staging/src/[k8s.io/client-go/util/workqueue/queue.go](http://k8s.io/client-go/util/workqueue/queue.go) 下的Type struct。
      3. ADD的操作是，如果加入新元素，发现这个新元素正在被处理时候，就不会重新加入，这样就避免了上面所说的问题。
   9. 

