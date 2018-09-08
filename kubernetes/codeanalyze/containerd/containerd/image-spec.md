# image spec

## image spec

image 有OCI 标准去维护， 链接地址 [https://github.com/opencontainers/image-spec/](https://github.com/opencontainers/image-spec/)

docker containerd 所使用的镜像都遵循这个标准， 大家都知道镜像是分层的， 从docker hub 拉取的镜像，基本上分为下面几个metadata：

![](../../../../.gitbook/assets/image.png)

  
image Index, image Mainfest, image config, layer

在containerd 下，默认这些文件都会存放在 

```text
/var/lib/containerd/io.containerd.content.v1.content/blobs/sha256
```

目录下



### image index

image index 是镜像最上层的数据文件， 主要表明了引用哪些menifest 每个menifest 代表了那个平台下的，是linux 还是amd 还是arm，docker 拉取镜像时候会先根据这个，来选择拉取哪个平台下的镜像

一个image index 会对应多个image mainfest



实例文件

```text
{
   "schemaVersion": 2,
   "mediaType": "application/vnd.docker.distribution.manifest.list.v2+json",
   "manifests": [
      {
         "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
         "size": 527,
         "digest": "sha256:5e8e0509e829bb8f990249135a36e81a3ecbe94294e7a185cc14616e5fad96bd",
         "platform": {
            "architecture": "amd64",
            "os": "linux"
         }
      },
      {
         "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
         "size": 527,
         "digest": "sha256:043b9724afca58fca70b1a95dfb5c88babad79c5e81242f8d42a05882a24158e",
         "platform": {
            "architecture": "arm",
            "os": "linux",
            "variant": "v5"
         }
      },
      {
         "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
         "size": 527,
         "digest": "sha256:6ba3c395f0a2941114d8dcdf80bedcc7e654252f5870dd94daff9cc3188f3eb2",
         "platform": {
            "architecture": "arm",
            "os": "linux",
            "variant": "v6"
         }
      },
      {
         "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
         "size": 527,
         "digest": "sha256:8819f5281d5356d94aaa61d9b3d5ececd86d05ca196642185c8e92894a656c66",
         "platform": {
            "architecture": "arm",
            "os": "linux",
            "variant": "v7"
         }
      },
      {
         "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
         "size": 527,
         "digest": "sha256:3316684e849b68e1a5e84e178b28fea36ea8a6a0e2863cb70e9cc334a659a0d0",
         "platform": {
            "architecture": "arm64",
            "os": "linux",
            "variant": "v8"
         }
      },
      {
         "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
         "size": 527,
         "digest": "sha256:32ed1bddef9423da80af5c29478ec64fc4dc643afe907287ca8444fe2ede8bd7",
         "platform": {
            "architecture": "386",
            "os": "linux"
         }
      },
      {
         "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
         "size": 528,
         "digest": "sha256:7005841571750707354a6443b5f9a31b751b3e08699d083dcc6a547369590c6d",
         "platform": {
            "architecture": "ppc64le",
            "os": "linux"
         }
      },
      {
         "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
         "size": 528,
         "digest": "sha256:5c5db4a7b21a622956dc06125e67ed65201957a38c2274cc36fe39f9264c7513",
         "platform": {
            "architecture": "s390x",
            "os": "linux"
         }
      }
   ]
}
```

### image mainfest

一个image mainfest 代表了这个平台下的镜像具体信息,主要包含两部分， image config 的指向，以及layers 的指向

以上面的 amd64 linux 平台的 

```text
5e8e0509e829bb8f990249135a36e81a3ecbe94294e7a185cc14616e5fad96bd
```

image mainfest 为例

实例内容

```text
{
   "schemaVersion": 2,
   "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
   "config": {
      "mediaType": "application/vnd.docker.container.image.v1+json",
      "size": 1497,
      "digest": "sha256:e1ddd7948a1c31709a23cc5b7dfe96e55fc364f90e1cebcde0773a1b5a30dcda"
   },
   "layers": [
      {
         "mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
         "size": 733241,
         "digest": "sha256:8c5a7da1afbc602695fcb2cd6445743cec5ff32053ea589ea9bd8773b7068185"
      }
   ]
}
```

可以看到 对应的config 文件是 

```text
e1ddd7948a1c31709a23cc5b7dfe96e55fc364f90e1cebcde0773a1b5a30dcda
```

只有一个layer 层

```text
8c5a7da1afbc602695fcb2cd6445743cec5ff32053ea589ea9bd8773b7068185
```

### image config

image config 文件保存了这个image 的cmd 命令， env 参数等等，按照



```text
e1ddd7948a1c31709a23cc5b7dfe96e55fc364f90e1cebcde0773a1b5a30dcda
```

为例， 打开内容如下

```text
{
  "architecture": "amd64",
  "config": {
    "Hostname": "",
    "Domainname": "",
    "User": "",
    "AttachStdin": false,
    "AttachStdout": false,
    "AttachStderr": false,
    "Tty": false,
    "OpenStdin": false,
    "StdinOnce": false,
    "Env": [
      "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
    ],
    "Cmd": [
      "sh"
    ],
    "ArgsEscaped": true,
    "Image": "sha256:c8f7edff56d0d88e50d01909c8c4824ccc98819f1f6b23e122d2678c472bf730",
    "Volumes": null,
    "WorkingDir": "",
    "Entrypoint": null,
    "OnBuild": null,
    "Labels": null
  },
  "container": "65b61a628c70e824d51fb7ac3f2ae5432b6fc52f812d0366d67df49a5e129571",
  "container_config": {
    "Hostname": "65b61a628c70",
    "Domainname": "",
    "User": "",
    "AttachStdin": false,
    "AttachStdout": false,
    "AttachStderr": false,
    "Tty": false,
    "OpenStdin": false,
    "StdinOnce": false,
    "Env": [
      "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
    ],
    "Cmd": [
      "/bin/sh",
      "-c",
      "#(nop) ",
      "CMD [\"sh\"]"
    ],
    "ArgsEscaped": true,
    "Image": "sha256:c8f7edff56d0d88e50d01909c8c4824ccc98819f1f6b23e122d2678c472bf730",
    "Volumes": null,
    "WorkingDir": "",
    "Entrypoint": null,
    "OnBuild": null,
    "Labels": {}
  },
  "created": "2018-07-31T22:20:07.617575594Z",
  "docker_version": "17.06.2-ce",
  "history": [
    {
      "created": "2018-07-31T22:20:07.361628468Z",
      "created_by": "/bin/sh -c #(nop) ADD file:96fda64a6b725d4df5249c12e32245e2f02469ff637c38077740f4984cd883dd in / "
    },
    {
      "created": "2018-07-31T22:20:07.617575594Z",
      "created_by": "/bin/sh -c #(nop)  CMD [\"sh\"]",
      "empty_layer": true
    }
  ],
  "os": "linux",
  "rootfs": {
    "type": "layers",
    "diff_ids": [
      "sha256:f9d9e4e6e2f0689cd752390e14ade48b0ec6f2a488a05af5ab2f9ccaf54c299d"
    ]
  }
}
```





### layer

layer 就是镜像的分层文件， 这是真正保存的每一层的更改，以

```text
8c5a7da1afbc602695fcb2cd6445743cec5ff32053ea589ea9bd8773b7068185
```

layer 为例， 我们可以把它解压

```text
tar zxvf 8c5a7da1afbc602695fcb2cd6445743cec5ff32053ea589ea9bd8773b7068185 -C bbb/
```

输出的内容如下（部分已经忽略）

```text
etc/network/
etc/network/if-down.d/
etc/network/if-post-down.d/
etc/network/if-pre-up.d/
etc/network/if-up.d/
etc/passwd
etc/shadow
home/
root/
tmp/
usr/
usr/sbin/
var/
var/spool/
var/spool/mail/
var/www/
```

关于layer 的制作标准链接[:https://github.com/opencontainers/image-spec/blob/master/image-layout.md](https://github.com/opencontainers/image-spec/blob/master/layer.md#applying)

layer 是一个tar 包

目标是

```text

A tar archive is then created which contains only this changeset:

Added and modified files and directories in their entirety
Deleted files or directories marked with a whiteout file
```

保存每一层新添加的文件， 修改的文件和删除的文件，

而删除的文件用.wh前缀来标注

