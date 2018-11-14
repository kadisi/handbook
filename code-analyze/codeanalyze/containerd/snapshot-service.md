# snapshot service

## Mount

Mount 的核心思想是要获得这个key 都需要mount 哪些目录 Mount 代码如下

```go
func (s *service) Mounts(ctx context.Context, mr *snapshotsapi.MountsRequest) (*snapshotsapi.MountsResponse, error) {
    log.G(ctx).WithField("key", mr.Key).Debugf("get snapshot mounts")
    sn, err := s.getSnapshotter(mr.Snapshotter)
    if err != nil {
        return nil, err
    }
    // sn 是  metadata/snapshot.go snapshotter struct
    mounts, err := sn.Mounts(ctx, mr.Key)
    if err != nil {
        return nil, errdefs.ToGRPC(err)
    }
    return &snapshotsapi.MountsResponse{
        Mounts: fromMounts(mounts),
    }, nil
}
```

{% hint style="info" %}
Super-powers are granted randomly so please submit an issue if you're not happy with yours.
{% endhint %}

Once you're strong enough, save the world:

```go
// Ain't no code for that yet, sorry 
echo 'You got to trust me on this, I saved the world'
```

