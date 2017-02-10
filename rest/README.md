### Doceph 卷插件的部署
#### (1) 安装软件doceph-daemon-ceph以及依赖包
```
[root@dev ~]# cp /path/to/docker-volume-ceph/rest/docker-daemon-ceph /opt/
[root@dev ~]# cp /path/to/docker-volume-ceph/contrib/upstart/rbdmap /usr/bin/
[root@dev ~]# chmod +x /opt/docker-daemon-ceph
[root@dev ~]# chmod +x /usr/bin/rbdmap*
[root@dev ~]# yum install oslo.config -y

```
编辑 /opt/docker-daemon-ceph,修改配置项:
```
ceph_pool
auth_user
default_volume_size
local_dir
default_file_system
volume_metadata
plugin_url_port
```

> **注意**

> 创建 local_dir 对应的路径,作为mountpoint(s)的父目录

> 部署ceph参考附录链接

> ceph_pool/auth_user 需要在ceph集群创建对应存储池以及具有rwx该池的用户

> default_volume_size 默认卷大小,单位GB

> default_file_system 创建卷之后默认格式化的文件系统

> 需要创建 volume_metadata 对应的路径,作为卷元数据(s)信息的父目录

> 修改 plugin_url_port 为当前宿主闲置端口,e.g:9999

#### (2) 启动服务
```
[root@dev ~]# nohup python /opt/doceph-daemon-local &
```
因为仅限本地docker daemon进程访问volume plugin daemon,所以 plugin_url_ip
为127.0.0.1,端口选择宿主某可用端口(此处以9999为例),且防火墙无需对外暴露端口.

添加docker daemon发现volume plugin daemon的endpoint
```
[root@dev ~]# mkdir -p /etc/docker/plugins/
[root@dev ~]# echo "http://127.0.0.1:9999" > /etc/docker/plugins/doceph.spec
```
#### (3) 重启docker服务,并观察volume plugin daemon日志
```
[root@dev ~]# docker volume ls
DRIVER              VOLUME NAME
doceph              volume-1q2xw340f
```
```
[root@dev ~]# tail -f nohup.out
127.0.0.1 - - [10/Feb/2017 01:31:47] "POST /Plugin.Activate HTTP/1.1" 200 32
127.0.0.1 - - [10/Feb/2017 01:31:47] "POST /VolumeDriver.List HTTP/1.1" 200 185
```

确认插件已经被激活
___
### Doceph 卷插件API测试

#### (1) 激活卷插件

请求:
```
[root@kvm-10216196 instance]# curl -X POST -d '{}' 10.160.0.144:8080/Plugin.Activate
```

响应:
```
{"Implements": ["VolumeDriver"]}
```

> **注意: **

> (1) 插件的发现与激活是由docker节点在本地自动完成的; 此处远程调用仅作测试;

> (2) kvm-10216196 是某远端服务器

> (3) 后续的 dev节点是docker计算节点服务器

#### (2) 创建卷
请求:
```
[root@kvm-10216196 instance]# curl -X POST -d '{"Name": "volume-1q2xw342f",\
"Opts":{"pool":"volumes","fs":"xfs","size":1,"auth_user":"cinder"}}' \
10.160.0.144:8080/VolumeDriver.Create
```

响应:
```
{"Err": ""}
```
查看创建结果:
```
[root@dev ~]# rbd showmapped | grep volume-1q2xw342f
0  volumes volume-1q2xw342f -    /dev/rbd0
```
```
[root@dev ~]# cat /etc/ceph/rbdmap | grep volume-1q2xw342f
volumes/volume-1q2xw342f id=cinder, keyring=AQCbbWxYPM5tBBAARTHNRPzuR5A2t/peKuFmIA==
```

```
[root@dev ~]# cat /etc/fstab | grep volume-1q2xw342f
/dev/rbd0 /data/docker-volumes/volume-1q2xw342f xfs defaults,_netdev 0 0
```

查看执行结果:
```
[root@dev ~]# rbd showmapped | grep volume-1q2xw342f
[root@dev ~]# cat /etc/ceph/rbdmap | grep volume-1q2xw342f
[root@dev ~]# cat /etc/ceph/rbdmap | grep volume-1q2xw342f
```
#### (3) 挂载卷
请求:
```
[root@kvm-10216196 instance]# curl -X POST -d '{"Name": "volume-1q2xw342f"}' 10.160.0.144:8080/VolumeDriver.Mount
```

响应:
```
{
    "Mountpoint": "/data/docker-volumes/volume-1q2xw342f",
    "Err": ""
}
```
#### (4) 卸载卷
请求:
```
[root@kvm-10216196 instance]# curl -X POST -d '{"Name": "volume-1q2xw342f"}' 10.160.0.144:8080/VolumeDriver.Unmount
```

响应:
```
{
    "Err": ""
}
```

#### (5) 删除卷
请求:
```
[root@kvm-10216196 instance]# curl -X POST -d '{"Name": "volume-1q2xw342f"}' 10.160.0.144:8080/VolumeDriver.Remove
```
```
{
    "Err": ""
}
```
#### (6) 查询卷信息
请求:
```
[root@kvm-10216196 instance]# curl -X POST -d '{"Name": "volume-1q2xw342f"}' 10.160.0.144:8080/VolumeDriver.Get
```

响应:
```
{
    "Volume": {
                  "Mountpoint": "/data/docker-volumes/volume-1q2xw342f",
                  "Name": "volume-1q2xw342f",
                  "Opts": {
                              "auth_user": "cinder",
                              "fs": "xfs",
                              "pool": "volumes",
                              "size": 1
                 }
              },
    "Err": ""
}
```
#### (7) 获取卷路径
请求:
```
[root@kvm-10216196 instance]# curl -X POST -d '{"Name": "volume-1q2xw342f"}' 10.160.0.144:8080/VolumeDriver.Path
```

响应:
```
{
    "Mountpoint": "/data/docker-volumes/volume-1q2xw342f",
    "Err": ""
}
```
#### (8) 列出插件管理的所有卷
请求:
```
[root@kvm-10216196 instance]# curl -X POST -d '{}' 10.160.0.144:8080/VolumeDriver.List
```

响应:
```
{
    "Err": "",
    "Volumes": [
                 {
                     "Mountpoint": "/data/docker-volumes/volume-1q2xw340f",
                     "Name": "volume-1q2xw340f",
                     "Opts": {
                               "auth_user": "cinder",
                               "fs": "xfs",
                               "pool": "volumes",
                               "size": 1
                     }
                 },
                 {
                     "Mountpoint": "/data/docker-volumes/volume-1q2xw342f", 
                     "Name": "volume-1q2xw342f",
                     "Opts": {
                                 "auth_user": "cinder",
                                 "fs": "xfs",
                                 "pool": "volumes","size": 1
                             }
                 }
               ]
}
```

### 附录
[使用ceph-deploy快速部署Ceph](https://tbd.com)

[Ceph客户端部署](https://github.com/guojy8993/blogs/blob/master/Ceph%E5%9D%97%E8%AE%BE%E5%A4%87%E5%AE%A2%E6%88%B7%E7%AB%AF%E7%9A%84%E9%83%A8%E7%BD%B2.md)

[映射Ceph RBD块设备到宿主](https://github.com/guojy8993/blogs/blob/master/Ceph%E5%9D%97%E8%AE%BE%E5%A4%87%E6%98%A0%E5%B0%84.md)
