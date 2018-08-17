# glusterfs集群部署接入k8s storageclass文档
简述：此次搭建的glusterfs集群是两个节点的glusterfs集群
10.1.0.37
10.1.0.77
如果搭建多节点的集群，在heketi挂盘操作的时候，需要更改topology.json
文件。

## 1.glusterfs部署
解压glusterfs安装包,安装glusterfs server端
```
tar -zxvf gluster-3.10-all.tar.gz
cd  gluster-3.10-all
rpm -Uvh  glusterfs*
```

## 2.启动glusterfs服务
```
sudo  service  glusterd    start
sudo  service  glusterfsd   start
```

## 3.添加glusterfs节点
```
gluster peer probe <each-address-of-other-glusterfs-machine>
gluster peer status    #查看glusterfs节点状态
```

## 4.heketi部署
```
sudo kubectl run heketi --image=docker.io/heketi/heketi:7 -n $namespaceName
```

## 5.更改heketi需要的配置文件
```
sudo mkdir config
sudo vi ./config/heketi.json
```
heketi.json文件如下：
``` 
{
  "_port_comment": "Heketi Server Port Number",
  "port": "8080",
 
  "_use_auth": "Enable JWT authorization. Please enable for deployment",
  "use_auth": true,
 
  "_jwt": "Private keys for access",
  "jwt": {
    "_admin": "Admin has access to all APIs",
    "admin": {
      "key": "kLd834dadEsfwcv"
    },
    "_user": "",
    "user": {
      "key": "Lkj763sdedsF"
    }
 
  },
 
  "_glusterfs_comment": "GlusterFS Configuration",
  "glusterfs": {
    "_executor_comment": [
      "Execute plugin. Possible choices: mock, ssh",
      "mock: This setting is used for testing and development.",
      "      It will not send commands to any node.",
      "ssh:  This setting will notify Heketi to ssh to the nodes.",
      "      It will need the values in sshexec to be configured.",
      "kubernetes: Communicate with GlusterFS containers over",
      "            Kubernetes exec api."
    ],
    "executor": "ssh",
 
    "_sshexec_comment": "SSH username and private key file information",
    "sshexec": {
      "keyfile": "/etc/heketi/heketi_key",
      "user": "root",
      "port": "22",
      "fstab": "/etc/fstab"
    },
 
    "_db_comment": "Database file name",
    "db": "/var/lib/heketi/heketi.db",
    "brick_min_size_gb": 1,
 
    "_loglevel_comment": [
      "Set log level. Choices are:",
      "  none, critical, error, warning, info, debug",
      "Default is warning"
    ],
    "loglevel" : "debug"
  }
}
```

## 6.生成heketi需要的私钥，公钥文件
```
sudo ssh-keygen -f ./config/heketi_key -t rsa -N ''
```

## 7.添加heketi的公钥文件内容到glusterfs节点的中
10.1.0.37和10.1.0.77都添加公钥
```
cat  ./config/heketi_pub.key  
vim  /root/.ssh/authorized_keys
```

## 8.准备topology.json文件，用来heketi挂卷
```
vim ./config/topology.json
```
topology.json文件如下：
``` 
{
    "clusters": [
        {
            "nodes": [
                {
                    "node": {
                        "hostnames": {
                            "manage": [
                                "10.1.0.37"
                            ],
                            "storage": [
                                "10.1.0.37"
                            ]
                        },
                        "zone": 1
                    },
                    "devices": [
                        "/dev/vdb"
                    ]
                },
                {
                    "node": {
                        "hostnames": {
                            "manage": [
                                "10.1.0.77"
                            ],
                            "storage": [
                                "10.1.0.77"
                            ]
                        },
                        "zone": 1
                    },
                    "devices": [
                        "/dev/vdb"
                    ]
                }
            ]
        }
    ]
}
```

## 9.创建secret
```
sudo kubectl create secret generic heketi-config --from-file=config/ -n $namespaceName
```

## 10.将secret挂到heketi的pod 里面
```
sudo kubectl edit deploy heketi -n heketi
```

## 11.格式化盘
要使用glusterfs客户端挂盘，要通过heketi来挂盘。
每个glusterfs machine上挂一快同样大小的未格式化过磁盘。
如果已经格式化过，使用下面任意一条命令重置磁盘(假设磁盘挂在/dev/vdc)：
建议选择第一条！！！！
```
sudo pvcreate --metadatasize=128M --dataalignment=256K /dev/vdc
sudo shred -v /dev/vdc
sudo dd if=/dev/zero of=/dev/vdc bs=1M
```

## 12.用heketi挂盘
```
heketi-cli --server=http://localhost:8080 --user=admin --secret=kLd834dadEsfwcv topology load --json=/etc/heketi/topology.json
```

## 13.查看结果
```
heketi-cli --server=http://localhost:8080 --user=admin --secret=kLd834dadEsfwcv topology info
```

## 14.以后也可以单独挂盘
```
heketi-cli --json --server=http://localhost:8080 --user=admin --secret=kLd834dadEsfwcv device add --name="/dev/vdd" --node="<node-id>"
```

## 15.创建storageclass
```
sudo kubectl create -f heketi_storageclass.yaml
```
storageclass编排文件如下：
```
apiVersion: storage.k8s.io/v1
kind: StorageClass  
metadata:
  name: heketi  
provisioner: kubernetes.io/glusterfs   
parameters:
  resturl: "http://10.1.0.71:31109"  
  restuser: "admin"  
  restuserkey: "kLd834dadEsfwcv"
  volumetype: "replicate:2"
```

## 16.创建pvc 测试
```
sudo kubectl create -f pvc.yaml -n heketi
```
pvc.yaml demo如下：
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
 name: pvc-test1
spec:
 accessModes:
  - ReadWriteMany
 resources:
   requests:
     storage: 1Gi
 storageClassName: heketi
```     
查看pvc状态
```
sudo kubectl get pvc -n heketi
sudo kubectl get pv
```
能够成功利用sc创建出pv，并且pvc状态为bound状态是成功状态
在heketi中查看brick是否生成
```
heketi-cli --server=http://localhost:8080 --user=admin --secret=kLd834dadEsfwcv topology info
```

## 17.删除pvc查看brick是否回收
```
sudo kubectl delete pvc $pvcname -n heketi
```
查看pv是否删除
```
sudo kubectl get pv
```
查看heketi中是否brick回收
```
heketi-cli --server=http://localhost:8080 --user=admin --secret=kLd834dadEsfwcv topology info
```
## 18.给heketi pod做持久化存储
heketi的数据存放在容器的/var/lib/heketi/下，db文件是heketi的元数据，container.logs是容器的申请的brick的日志数据。
首先利用heketi创建出pvc pv
```
sudo kubectl create -f heketi_pvc.yaml -n heketi
```
把heketi pod中的元数据cp 到本地一份(一定要保存出来，否则一会挂pvc数据就没有了)
```
sudo kubectl cp heketi/$heketipodname:/var/lib/heketi/* .
```
把pvc挂到heketi pod中的/var/lib/heketi/下。
heketi的deploy编排如下：
``` 
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  annotations:
    deployment.kubernetes.io/revision: "4"
  generation: 4
  labels:
    run: heketi
  name: heketi
  selfLink: /apis/extensions/v1beta1/namespaces/heketi/deployments/heketi
spec:
  replicas: 1
  selector:
    matchLabels:
      run: heketi
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
    type: RollingUpdate
  template:
    metadata:
      creationTimestamp: null
      labels:
        run: heketi
    spec:
      containers:
      - image: 10.1.0.159/heketi/heketi:7
        imagePullPolicy: IfNotPresent
        name: heketi
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
        volumeMounts:
        - mountPath: /etc/heketi/
          name: heketi-config
        - mountPath: /var/lib/heketi/
          name: heketi-db
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30
      volumes:
      - name: heketi-config
        secret:
          defaultMode: 420
          secretName: heketi-config
      - name: heketi-db
        persistentVolumeClaim:
          claimName: heketi-db
```
## 19.再次创建pvc验证heketi pod是否功能正常
执行步骤15
