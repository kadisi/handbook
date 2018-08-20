# 调度选举

## 简介

选举原理 scheduler 使用了两种方式来进行master 选举 : endpoint, 和 configmap， 以endpoint 为例



1.  endpoint  多个 scheduler 启动时候， scheduler默认会在kube-system 下创建一个kube-scheduler 的endpoint，首先创建到的则就是master，剩下的scheduler ，会一直get 这个ep，一直到发现ep里的注册信息超时了，则会开始抢占，更新这个endpoint，谁先抢占成功，则谁就是master，执行sheduler 的真正操作
2. 但是 会存在，当一个master 死掉后，两个slave 在同一个周期内发现master 死掉，超时，在同一个周期内，都会执行update 操作，当都update 成功后，都会认为自己是master，会执行真正的scheduler 操作，但是，shceduler 还有一个协程，renew（） 方法，默认每隔2秒钟进行一次检查，当检查到自己不是真正的master 时候，会stop scheduler 操作

