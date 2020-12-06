### [Install Docker Engine on CentOS](https://docs.docker.com/engine/install/centos/)

#### iptables Reset
```shell
iptables -Z # zero counters
iptables -F # flush (delete) rules
iptables -X # delete all extra chains
```

#### Disable Firewalld
```shell
sudo systemctl stop firewalld
sudo systemctl disable firewalld
```

#### Uninstall old versions
```shell
sudo yum remove docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-engine
```

#### Remove Old Contents
```shell
rm -rf /var/lib/docker
```

#### Set up the Repository
```shell
sudo yum install -y yum-utils

sudo yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo
```

#### Install Docker Engine
- Install the latest version of Docker Engine and containerd, or go to the next step to install a specific version:
  ```shell
  sudo yum install docker-ce docker-ce-cli containerd.io
  ```

- To install a specific version of Docker Engine, list the available versions in the repo, then select and install:
  ```shell
  yum list docker-ce --showduplicates | sort -r
  
  docker-ce.x86_64  3:18.09.1-3.el7                     docker-ce-stable
  docker-ce.x86_64  3:18.09.0-3.el7                     docker-ce-stable
  docker-ce.x86_64  18.06.1.ce-3.el7                    docker-ce-stable
  docker-ce.x86_64  18.06.0.ce-3.el7                    docker-ce-stable
  ```

  ```shell
  # sudo yum install -y docker-ce-3:18.09.9-3.el7 docker-ce-cli-1:19.03.14-3.el7 containerd.io
  sudo yum install docker-ce-18.0* containerd.io
  
  sudo systemctl start docker
  
  # Set up the Docker daemon (https://kubernetes.io/docs/setup/production-environment/container-runtimes/#docker)
  cat <<EOF | sudo tee /etc/docker/daemon.json
  {
    "exec-opts": ["native.cgroupdriver=systemd"],
    "log-driver": "json-file",
    "log-opts": {
      "max-size": "100m"
    },
    "storage-driver": "overlay2",
    "storage-opts": [
      "overlay2.override_kernel_check=true"
    ]
  }
  EOF
  
  sudo systemctl enable docker
  sudo systemctl status docker -l
  ```


### [Installing kubeadm](https://v1-18.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)


#### Letting iptables see bridged traffic
Make sure that the br_netfilter module is loaded. This can be done by running lsmod | grep br_netfilter. To load it explicitly call sudo modprobe br_netfilter.

As a requirement for your Linux Node's iptables to correctly see bridged traffic, you should ensure `net.bridge.bridge-nf-call-iptables` is set to 1 in your `sysctl` config,   
e.g.

```shell
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF

sudo sysctl --system
```


#### Installing kubeadm, kubelet and kubectl
```shell
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-\$basearch
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
exclude=kubelet kubeadm kubectl
EOF

# Set SELinux in permissive mode (effectively disabling it)
sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config

sudo yum install -y kubelet-1.18.9-0 kubeadm-1.18.9-0 kubectl-1.18.9-0 --disableexcludes=kubernetes

sudo systemctl enable --now kubelet
```


### [Creating a cluster with kubeadm](https://v1-18.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/)

#### (master) kubeadm init
```shell
kubeadm init --kubernetes-version=v1.18.9 \
--pod-network-cidr=10.0.0.0/16 \
--service-cidr=192.168.25.0/26 \
--v=3 
```   

<details>
  <summary>HA 세팅시에 참고할 내용</summary>
[Customizing control plane configuration with kubeadm](https://v1-18.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/control-plane-flags/)

The kubeadm ClusterConfiguration object exposes the field extraArgs that can override the default flags passed to control plane components such as the APIServer, ControllerManager and Scheduler. The components are defined using the following fields:
- apiServer
- controllerManager
- scheduler

The extraArgs field consist of `key: value` pairs. To override a flag for a control plane component:

1. Add the appropriate fields to your configuration.
2. Add the flags to override to the field.
3. Run `kubeadm init` with `--config <YOUR CONFIG YAML>.`


</details>
---


To make kubectl work for your `non-root user`, run these commands, which are also part of the kubeadm init output:
```shell
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Alternatively, if you are the root user, you can run:
```shell
export KUBECONFIG=/etc/kubernetes/admin.conf
```



### [Installing a Pod network add-on](https://v1-18.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/#pod-network)

#### [Install Calico](https://docs.projectcalico.org/getting-started/kubernetes/quickstart)
*CNI를 설치하고 난 후에야 노드조인이 가능하다.*

```shell
wget -O custom-resource.yaml https://docs.projectcalico.org/manifests/custom-resources.yaml
```


### worker join
```shell
kubeadm join 192.168.25.30:6443 --token z7cxbb.2d1n4vc3l2t4decy \
    --discovery-token-ca-cert-hash sha256:b5b124d4c4816d8dcccb88197aac4cae69a5bdf6acb2e1c46c4305a205fa30a6
```



---
### Node Join Fail
- clean worker
```shell
# join fail
[root@worker01 ~]# kubeadm join 192.168.25.30:6443 --token kt69ja.jsi51tf1224i7zll \
>     --discovery-token-ca-cert-hash sha256:33b0bada7baa4c3a0e284c59f5182c2c5de58da44a2e06aa4e8206833fb723d6
W1207 01:08:08.824481   31282 join.go:346] [preflight] WARNING: JoinControlPane.controlPlane settings will be ignored when control-plane flag is not set.
[preflight] Running pre-flight checks
        [WARNING IsDockerSystemdCheck]: detected "cgroupfs" as the Docker cgroup driver. The recommended driver is "systemd". Please follow the guide at https://kubernetes.io/docs/setup/cri/
error execution phase preflight: [preflight] Some fatal errors occurred:
        [ERROR FileAvailable--etc-kubernetes-kubelet.conf]: /etc/kubernetes/kubelet.conf already exists
        [ERROR Port-10250]: Port 10250 is in use
        [ERROR FileAvailable--etc-kubernetes-pki-ca.crt]: /etc/kubernetes/pki/ca.crt already exists
[preflight] If you know what you are doing, you can make a check non-fatal with `--ignore-preflight-errors=...`
To see the stack trace of this error execute with --v=5 or higher


# 에러 처리
[root@worker01 ~]# lsof -i:10250
COMMAND PID USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
kubelet 813 root   30u  IPv6  23876      0t0  TCP *:10250 (LISTEN)

# 사용중인 프로세스 제거
kill -9 813

# 이전 설정파일 제거
rm -rf /etc/kubernetes/kubelet.conf /etc/kubernetes/pki/ca.crt

iptables -Z # zero counters
iptables -F # flush (delete) rules
iptables -X # delete all extra chains
```
