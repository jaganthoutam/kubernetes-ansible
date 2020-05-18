# kubernetes-ansible
Ansible playbook to create a Highly Available kubernetes cluster using release (1.16.9) on
Bare metal system (have been tested on CentOS 7, Ubuntu 18.04).

Requirements:
 - Ansible 2.7

Firewalld for Master 
```bash
systemctl restart firewalld
systemctl enable firewalld
firewall-cmd --permanent --zone=public --add-port=80/tcp
firewall-cmd --permanent --zone=public --add-port=8081/tcp
firewall-cmd --permanent --zone=public --add-port=443/tcp
firewall-cmd --permanent --zone=public --add-port=6443/tcp
firewall-cmd --permanent --zone=public --add-port=9090/tcp
firewall-cmd --permanent --zone=public --add-port=2379-2380/tcp
firewall-cmd --permanent --zone=public --add-port=10250/tcp
firewall-cmd --permanent --zone=public --add-port=10251/tcp
firewall-cmd --permanent --zone=public --add-port=10252/tcp
firewall-cmd --permanent --zone=public --add-port=10255/tcp
firewall-cmd --permanent --zone=public --add-port=8472/udp
firewall-cmd --zone=public --add-masquerade --permanent
# only if you want NodePorts exposed on control plane IP as well
firewall-cmd --permanent --zone=public --add-port=30000-32767/tcp
firewall-cmd --reload
systemctl restart firewalld
```

Firewalld for Worker
```bash
systemctl restart firewalld
systemctl enable firewalld
firewall-cmd --permanent --zone=public --add-port=80/tcp
firewall-cmd --permanent --zone=public --add-port=443/tcp
firewall-cmd --zone=public --permanent --add-port=10250/tcp
firewall-cmd --zone=public --permanent --add-port=10255/tcp
firewall-cmd --zone=public --permanent --add-port=8472/udp
firewall-cmd --zone=public --permanent --add-port=30000-32767/tcp
firewall-cmd --zone=public --add-masquerade --permanent
firewall-cmd --reload
systemctl restart firewalld
```


Download the Kubernetes-Ansible playbook and set up variable according to need in group variable file
[all.yml](group_vars/all.yml.example). Please read this file carefully and modify according to your need.

```
git clone https://github.com/pawankkamboj/kubernetes-ansible.git
cd kubernetes-ansible
cp group_vars/all.yml.example group_vars/all.yml
cp inventory.example inventory
ansible-playbook -i inventory -b -u root cluster.yml 
```
 * Kubectl commands should be run on Master nodes only.

After magic happened Login to one of the master node and we should test Kubernetes cluster installation by executing `kubectl --kubeconfig=/etc/kubernetes/kubeadminconfig get nodes'` command.
After a while (5 mins or more) we have to see this result:
```
NAME          STATUS     ROLES    AGE    VERSION
gt-crc-s157   NotReady   <none>   106m   v1.18.1
gt-crc-s158   NotReady   <none>   48m    v1.18.1
gt-crc-s159   NotReady   <none>   48m    v1.18.1
gt-crc-s170   NotReady   <none>   47m    v1.18.1
gt-crc-s171   NotReady   <none>   47m    v1.18.1
```
It is showing `NotReady` because kubernetes network plugin is not deployed.

We are almost there, only addons left.
```
ansible-playbook -i inventory -b -u root addon.yml
```
Test result of execution with `kubectl --kubeconfig=/etc/kubernetes/kubeadminconfig get pods -n kube-system'` command.
```
NAME                                     READY   STATUS    RESTARTS   AGE
coredns-54f546d556-6tqfv                 1/1     Running   0          154m
etcd-gt-crc-s157                         1/1     Running   0          3h36m
etcd-gt-crc-s158                         1/1     Running   0          158m
etcd-gt-crc-s159                         1/1     Running   0          158m
kube-apiserver-gt-crc-s157               1/1     Running   0          3h36m
kube-apiserver-gt-crc-s158               1/1     Running   0          158m
kube-apiserver-gt-crc-s159               1/1     Running   0          158m
kube-controller-manager-gt-crc-s157      1/1     Running   0          3h37m
kube-controller-manager-gt-crc-s158      1/1     Running   0          158m
kube-controller-manager-gt-crc-s159      1/1     Running   0          157m
kube-flannel-ds-amd64-fscwr              1/1     Running   0          65m
kube-flannel-ds-amd64-hz9cr              1/1     Running   0          65m
kube-flannel-ds-amd64-l7rxl              1/1     Running   0          65m
kube-flannel-ds-amd64-rw762              1/1     Running   0          65m
kube-flannel-ds-amd64-sjw9x              1/1     Running   0          65m
kube-proxy-8fbgm                         1/1     Running   0          154m
kube-proxy-8hvmv                         1/1     Running   0          154m
kube-proxy-gwtcp                         1/1     Running   0          154m
kube-proxy-m8s24                         1/1     Running   0          154m
kube-proxy-rw6wn                         1/1     Running   0          154m
kube-scheduler-gt-crc-s157               1/1     Running   0          3h37m
kube-scheduler-gt-crc-s158               1/1     Running   0          158m
kube-scheduler-gt-crc-s159               1/1     Running   0          158m
metrics-server-v0.3.6-8448866767-qv9nr   1/1     Running   0          154m

```
 # setup KUBECONFIG 
 Run below commands on all the server
 ```bash
export KUBECONFIG=/etc/kubernetes/kubeadminconfig
 
[root@GT-CRC-S157 net.d]# kubectl get no
NAME          STATUS   ROLES    AGE     VERSION
gt-crc-s157   Ready    <none>   3h41m   v1.18.1
gt-crc-s158   Ready    <none>   162m    v1.18.1
gt-crc-s159   Ready    <none>   162m    v1.18.1
gt-crc-s170   Ready    <none>   161m    v1.18.1
gt-crc-s171   Ready    <none>   161m    v1.18.1
 
 ```
 
 


Ansible roles
- yum-proxy - installs Squid proxy server
- yum-repo - installs epel repo
- sslcert - create all ssl certificates require to run secure K8S cluster
- runtime-env - common settings for container runtime environment
- docker - install latest docker release on all cluster nodes
- containerd - IF you want to use containerd runtime instead of Docker, use this role and enable this in group variable file
- etcd - setup etcd cluster, running as container on all master nodes
- haproxy - setup haproxy for API service HA, running on all master nodes
- keepalived - using keepalive for HA of IP address for kube-api server, running on all master nodes
- master - setup kubernetes master controller components - `kube-apiserver`, `kube-controller`, `kube-scheduler`, `kubectl` client
- node - setup kubernetes node agent - `kubelet`
- addon - to create addon services: `flanneld`, `kube-proxy`, `kube-dns`, `metrics-server`

Note - Addon roles should be run after cluster is fully operational. Addons are in [addons.yml](addons.yml) playbook.
After cluster is up and running then run `addon.yml` to deploy add-ons.
Included addons are: `flannel network`, `kube-proxy`, `coredns`.
```
ansible-playbook -i inventory addon.yml
```


## kubernetes HA architecture
Below is a sample Kubernetes cluster architecture after successfully building it using playbook. It is just a sample, so number of servers/node may vary according to your setup.

![kubernetes HA architecture](kubernetes_architecture.png)
