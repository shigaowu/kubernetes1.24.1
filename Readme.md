# k8s 1.23.1升级到1.24.1
## 一、从docker迁移到container
###  1、腾空节点
   ``` kubectl drain k8s-master01 --ignore-daemonsets ```
### 2、停止docker守护进程
   ``` systemctl stop kubelet ```
   ``` systemctl disable docker.service --now ```
### 3、安装container
#### 下载安装包:
``` wget https://github.com/containerd/containerd/releases/download/v1.6.4/containerd-1.6.4-linux-amd64.tar.gz ```
#### 解压安装包:
``` tar Cxzvf /usr/local containerd-1.6.4-linux-amd64.tar.gz ```
#### 创建服务:
```
vi /usr/lib/systemd/system/containerd.service
[Unit]
Description=containerd container runtime
Documentation=https://containerd.io
After=network.target local-fs.target

[Service]
ExecStartPre=-/sbin/modprobe overlay
ExecStart=/usr/local/bin/containerd

Type=notify
Delegate=yes
KillMode=process
Restart=always
RestartSec=5
# Having non-zero Limit*s causes performance problems due to accounting overhead
# in the kernel. We recommend using cgroups to do container-local accounting.
LimitNPROC=infinity
LimitCORE=infinity
LimitNOFILE=infinity
# Comment TasksMax if your systemd version does not supports it.
# Only systemd 226 and above support this version.
TasksMax=infinity
OOMScoreAdjust=-999

[Install]
WantedBy=multi-user.target
```
#### 配置 containerd：
   ```sudo mkdir -p /etc/containerd```
   ```containerd config default | sudo tee /etc/containerd/config.toml```
注意:这里要改一下pause的域名，不然镜像拉不下来kubelet就起不来，即使你提前准备好镜像打同样的tag都不行。
```sed -i '/sandbox_image/s/k8s.gcr.io\/pause:3.6/registry.aliyuncs.com\/google_containers\/pause:3.6/' /etc/containerd/config.toml```
#### 重启 containerd：
   ```sudo systemctl restart containerd```

这时候再get node查看信息可以看到CONTAINER-RUNTIME变成containerd://1.6.4了，再也不是熟悉的docker://20.10.6了



## 二、升级到1.24.1
### 1、升级kubeadm到1.24.1
   ```yum install -y kubeadm-1.24.1-0 --disableexcludes=Kubernetes```

用kubeadm升级的时候:
对于使用 kubeadm 的用户，可以考虑下面的问题：
kubeadm 工具将每个主机的 CRI 套接字保存在该主机对应的 Node 对象的注解中。 使用 kubeadm 的用户应该知道，kubeadm 工具将每个主机的 CRI 套接字保存在该主机对应的 Node 对象的注解中。 要更改这一注解
信息，你可以在一台包含 kubeadm /etc/kubernetes/admin.conf 文件的机器上执行以下命令：
kubectl edit no k8s-master01
这一命令会打开一个文本编辑器，供你在其中编辑 Node 对象。 要选择不同的文本编辑器，你可以设置 KUBE_EDITOR 环境变量。
更改 kubeadm.alpha.kubernetes.io/cri-socket 值，将其从 /var/run/dockershim.sock 改为你所选择的 CRI 套接字路径 （例如：unix:///run/containerd/containerd.sock）。注意新的 CRI 套接字路径必须带有
 unix:// 前缀。
保存文本编辑器中所作的修改，这会更新 Node 对象。

### 2、升级到的目标版本
   ```kubeadm upgrade apply v1.24.1```

### 3、腾空节点:
   ```kubectl drain k8s-master01  --ignore-daemonsets```

### 4、升级kubelet和kubectl
   ```yum install -y kubelet-1.24.1-0 kubectl-1.24.1-0 --disableexcludes=Kubernetes```

### 5、重启kubelet
这里要注意!
   ```vi /var/lib/kubelet/kubeadm-flags.env```
删掉 --network-plugin参数，这个参数已经过时了。
重启kubelet
不然会报错
```Jun 01 18:38:39 k8s-master01 kubelet[3499]: Error: failed to parse kubelet flag: unknown flag: --network-plugin```

```
systemctl daemon-reload
systemctl restart kubelet
```

### 6、解除节点保护:
```kubectl uncordon k8s-master01```
