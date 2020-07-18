# Ansible脚本部署Kubernetes v1.18 高可用集群
### 一、安装要求  

在开始之前，部署Kubernetes集群机器需要满足以下几个条件：

- 一台或多台机器，操作系统 CentOS7.x-86_x64
- 硬件配置：2GB或更多RAM，2个CPU或更多CPU，硬盘30GB或更多
- 集群中所有机器之间网络互通
- 可以访问外网，需要拉取镜像，如果服务器不能上网，需要提前下载镜像并导入节点
- 禁止swap分区

确保所有节点系统时间一致

```shell
# yum install ntpdate -y
# ntpdate time.windows.com
```

**单Master架构图**

![single-master](/Users/calvin/Desktop/ansible-install-k8s/assets/single-master.jpg)

**单Master服务器规划**

|  节点名称  |     IP      |
| :--------: | :---------: |
|   etcd-1   | 10.10.10.8  |
|   etcd-2   | 10.10.10.9  |
| k8s-node02 | 10.10.10.10 |



### 二、下载所需文件并修改配置

#### 1、下载Ansible部署文件

```shell
# git clone https://github.com/calvinGithub/ansible-install-k8s
# cd ansible-install-k8s
```

下载软件包并解压/root目录：

链接:https://pan.baidu.com/s/11-c6ZEwwKS2YsnZqlcMIyw  密码:gsep
```
# tar zxf binary_pkg.tar.gz
```


#### 2、修改Ansible文件

修改hosts文件，根据规划修改对应IP和名称。

```shell
# vi hosts

[master]
# 如果部署单Master，只保留一个Master节点
# 默认Naster节点也部署Node组件
10.10.10.8 node_name=k8s-master01

[node]
10.10.10.9 node_name=k8s-node01
10.10.10.10 node_name=k8s-node02

[etcd]
10.10.10.8 etcd_name=etcd-1
10.10.10.9 etcd_name=etcd-2
10.10.10.10 etcd_name=etcd-3

[lb]
# 如果部署单Master，该项忽略
192.168.31.63 lb_name=lb-master
192.168.31.71 lb_name=lb-backup

[k8s:children]
master
node

[newnode]
#192.168.31.91 node_name=k8s-node3
```


#### 3、修改group_vars/all.yml文件

修改软件包目录和证书可信任IP。

```shell
# vim group_vars/all.yml

# 安装目录 
software_dir: '/root/binary_pkg'
k8s_work_dir: '/opt/kubernetes'
etcd_work_dir: '/opt/etcd'
tmp_dir: '/tmp/k8s'

# 集群网络
service_cidr: '10.0.0.0/24'
cluster_dns: '10.0.0.2'   # 与roles/addons/files/coredns.yaml中IP一致，并且是service_cidr中的IP
pod_cidr: '10.244.0.0/16' # 与roles/addons/files/kube-flannel.yaml中网段一致
service_nodeport_range: '30000-32767'
cluster_domain: 'cluster.local'

# 高可用，如果部署单Master，该项忽略
vip: '192.168.31.88'
nic: 'ens33' 

# 自签证书可信任IP列表，为方便扩展，可添加多个预留IP
cert_hosts:
  # 包含所有LB、VIP、Master IP和service_cidr的第一个IP
  k8s:
    - 10.0.0.1
    - 10.10.10.8
    - 10.10.10.9
    - 10.10.10.10
  # 包含所有etcd节点IP
  etcd:
    - 10.10.10.8
    - 10.10.10.9
    - 10.10.10.10
```


### 三、一键部署

#### 1、centos7安装Ansible

```shell
# yum install -y epel-release
# yum install ansible -y
```



#### 2、单Master版启动命令

```shell
# ansible-playbook -i hosts single-master-deploy.yml -uroot -k
```
命令结束后，可以看到生成了访问令牌

![image-20200718230920066](/Users/calvin/Desktop/ansible-install-k8s/assets/image-20200718230920066.png)

```shell
eyJhbGciOiJSUzI1NiIsImtpZCI6IjFhMTN5LXdUVFN6TVVoNmZ1dUltVngyTmxvMXNKejFTYkZkbldaNWhWdXcifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlcm5ldGVzLWRhc2hib2FyZCIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJkYXNoYm9hcmQtYWRtaW4tdG9rZW4tcXRyNmwiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC5uYW1lIjoiZGFzaGJvYXJkLWFkbWluIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQudWlkIjoiYWY5YWIyNmMtN2MwZC00MTZkLWJhYmMtZTMwMWY5MDBmMjU0Iiwic3ViIjoic3lzdGVtOnNlcnZpY2VhY2NvdW50Omt1YmVybmV0ZXMtZGFzaGJvYXJkOmRhc2hib2FyZC1hZG1pbiJ9.4gRDjg8It_8h0LInLHXrchsUCcdH0UerDDkCdd2y_UnKwc6wAhReDQYc2NaYVda2704Hu7-n7_fXFATQ_bf6noBjoQNsCEJ1NwBl0DO4h3l_i80YIVT9R2yl1nEq6CY5Da7rR01pp6-Htih1mpEb-O_xsP_L6FrsVjKBubY63gK4O9LQLQSnoEE-24ggg2VgyS5rVHTTDlEpq3J0XeqXiBbaZJa8o65Yg08sKzeHAffgY2538p-9ZV5-tJAsL4wbud8TjSXCzWkxsVsmH7Au0ZwRrth1tVrVEqoPGOusDVu6sFxSO53lpwsvlcg1TpKIxNMEbDTQ6GSJbPA_dyeW9Q
```



### 四、测试部署情况

#### 1、访问Dashboard

随便访问任何一个节点，https://Node:30001

![image-20200718231233862](/Users/calvin/Desktop/ansible-install-k8s/assets/image-20200718231233862.png)

进入页面后，点击左侧导航栏Nodes，可以看到目前加入k8s的服务器节点。

![image-20200718231355987](/Users/calvin/Desktop/ansible-install-k8s/assets/image-20200718231355987.png)



#### 2、使用kubectl命令查看容器

```shell
# 1.查看nodes
[root@k8s-master01 ansible-install-k8s]# kubectl get node
NAME           STATUS   ROLES    AGE   VERSION
k8s-master01   Ready    <none>   15m   v1.18.6
k8s-node01     Ready    <none>   15m   v1.18.6
k8s-node02     Ready    <none>   15m   v1.18.6

# 2.创建应用nginx
[root@k8s-master01 ansible-install-k8s]# kubectl create deployment web --image=nginx
deployment.apps/web created

# 3.启动应用nginx
[root@k8s-master01 ansible-install-k8s]# kubectl expose deployment web --port=80 --target-port=80 --name=web --type=NodePort
service/web exposed

# 4.查看所有启动的应用(30794即为容器80端口对应的宿主机映射端口)
[root@k8s-master01 ansible-install-k8s]# kubectl get svc
NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)        AGE
kubernetes   ClusterIP   10.0.0.1     <none>        443/TCP        32m
web          NodePort    10.0.0.95    <none>        80:30794/TCP   38s
```



#### 3、Dashboard查看运行容器

![image-20200718232547965](/Users/calvin/Desktop/ansible-install-k8s/assets/image-20200718232547965.png)

可以看到测试用的web容器已经启动，查看容器日志

![image-20200718232658162](/Users/calvin/Desktop/ansible-install-k8s/assets/image-20200718232658162.png)

浏览器访问：http://10.10.10.9:30794/

![image-20200718232858065](/Users/calvin/Desktop/ansible-install-k8s/assets/image-20200718232858065.png)



至此，单Master的k8s集群及部署docker应用已经成功。