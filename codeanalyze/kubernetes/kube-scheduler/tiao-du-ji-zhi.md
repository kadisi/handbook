# 调度机制



## kube-scheduler 的调度机制



1. 在pod 亲和和反亲和中，**requiredDuringSchedulingIgnoredDuringExecution** 是在预选一步操作的  **preferredDuringSchedulingIgnoredDuringExecution** 是在优选一步实现的
2.  PreferredDuringScheduling 不管是亲和还是反亲和 都是对称的
3.  RequiredDuringScheduling 的反亲和是对称的     RequiredDuringScheduling  的亲和是不对称的



