# First go at Kubernetes

This is my first run at Kubernetes.

## The Setup

So I have my own hardware in a DC. Weird flex, I know people are sick of hearing about it, but it's a childhood dream so shut up and let me nerd out about it.

I have three VM's running on my hypervisor; ed, edd and eddie. I didn't realize it was "Eddy" and not "Eddie". These things happen.

Ed and Eddie are both workers. Edd is a master. Fans of the show will get it.

Edd has the control plane. Kubeadm is a control node.

## Ansible

I avoided the Kubernetes Ansible plugin initially because I wanted to learn from scratch. I wasn't bothered to manually configure three machines though.

I have three roles for this: one role for all nodes, one for the master and one for the worker. Currently, there's no intelligence to handle switching the master.

j7b-k8: Install required packages, upload configurations, ensure services are running, create 'normal' users to avoid running things as root.

j7b-k8-master: Nothing initially; created control plane by hand

j7b-k8-worker: Nothing initially; joined control plane by hand

### Ansible Configuration

*/etc/ansible/hosts*

```
[k8-master]
edd

[k8-worker]
ed
eddie
```

*/etc/ansible/playbooks/k8.yml*

```
---
- hosts:
    - k8-master
    - k8-worker
  roles:
    - default
    - j7b-k8

- hosts:
    - k8-master
  roles:
    - j7b-k8-master

- hosts:
    - k8-worker
  roles:
    - j7b-k8-worker
```

*roles*

My roles can be picked up from Github. There are three:

- j7b-k8
- j7b-k8-master
- j7b-k8-worker

Note that the 'default' role is simply my default config that I push to all nodes. This does stuff like key management.

### Basic Steps 

Ansible does the following, in no particular order:

1. Configure Kubernetes repo
2. Create the 'icekube' user and group, and configures it's .kube config dir
3. Changed Docker cgroup from cgroupfs to systemd
4. Configures firewalld for Kubernetes
5. Disables swap
6. Copies over some basic configs from the kubernetes docs
7. Install Kubectl, Kubeadm and Kubelet

You are then left to configure the cluster yourself.

## Create the Control Plane

ssh onto edd and run the following command:

```
kubeadm init --pod-network-cidr=10.8.0.0/16
```

As I was testing out the above and building the playbook, I would run into errors and kubeadm would partially init the environment. My favorite feature about kudeadm is that it has `kubeadm reset --force` which reverts your system back to a factory state. 

The init script will do its thing and finally spit out the following information:

```
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join <edds ip>:6443 --token <token> \
        --discovery-token-ca-cert-hash <some hash>
```

What you need to do is literally told to you. We've already created the "regular" user (icekube), so (as root):

```
cp /etc/kubernetes/admin.conf /home/icekube/.kube/config
ln -s /etc/kubernetes/admin.conf /root/.kube/config
chown icekube:icekube /home/icekube/.kube/admin.conf
```

`kubectl` looks at ~/.kube/config to know how to connect to the api. If this is not configured, you'll get a no-response to `localhost:8080` message, which appears to be the default port `kubectl` uses.

As glossed over in official docs, you should be able to run the following to get a similar result. Note "pending" and "NotReady": *this is normal*. We need to install the networking addon.

```
[root@edd ~]# kubectl get pods -n kube-system
NAME                          READY   STATUS    RESTARTS   AGE
coredns-78fcd69978-k5sbq      0/1     Pending   0          95m
coredns-78fcd69978-xzqhv      0/1     Pending   0          95m
etcd-edd                      1/1     Running   1          95m
kube-apiserver-edd            1/1     Running   1          95m
kube-controller-manager-edd   1/1     Running   1          95m
kube-proxy-82xhn              1/1     Running   0          56m
kube-proxy-8t4kg              1/1     Running   0          95m
kube-proxy-dz6hp              1/1     Running   0          56m
kube-scheduler-edd            1/1     Running   1          95m
[root@edd ~]# kubectl get nodes
NAME    STATUS     ROLES                  AGE    VERSION
edd     NotReady   control-plane,master   105m   v1.22.1
```

I have opted for Calico as it seems to be a go-to in the community. When I first did this, I joined my worker nodes before I installed Calico. So we'll do Calico's config in two sections time; keep reading.

For the record, my real `kubectl get nodes` is:

```
[root@edd ~]# kubectl get nodes
NAME    STATUS     ROLES                  AGE    VERSION
ed      NotReady   <none>                 66m    v1.22.1
edd     NotReady   control-plane,master   105m   v1.22.1
eddie   NotReady   <none>                 66m    v1.22.1
```

## Clustering in Workers

I'm somewhat lazy here...

```
ansible -m shell \
    -a "kubeadm join <edds ip>:6443 --token <token> --discovery-token-ca-cert-hash <some hash>" \
    --become-user icekube \
    k8-workers
```

I ran this presuming it would work and kind of hoped it failed so I had something more to learn. Unfortunately it worked just fine, see below:

```
<ed ip> | CHANGED | rc=0 >>
[preflight] Running pre-flight checks
[preflight] Reading configuration from the cluster...
[preflight] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Starting the kubelet
[kubelet-start] Waiting for the kubelet to perform the TLS Bootstrap...

This node has joined the cluster:
* Certificate signing request was sent to apiserver and a response was received.
* The Kubelet was informed of the new secure connection details.

Run 'kubectl get nodes' on the control-plane to see this node join the cluster. [WARNING FileExisting-tc]: tc not found in system path
```


## Something Something Install calico

Firstly, I didn't have port 179/tcp open; do this if you haven't. I have updated my role to reflect this.

Project Calico tells you to apply their yaml directly from their server. **Don't do this**. Do the following instead:

```
[root@edd ~]# curl https://docs.projectcalico.org/manifests/calico.yaml -O calico.yaml
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  197k  100  197k    0     0   189k      0  0:00:01  0:00:01 --:--:--  189k
```

Edit `calico.yaml` and modify `CALICO_IPV4POOL_CIDR` to suit your needs. I'm personally running my containers in `10.80.0.0/16` as I'm a fiend for a well organized network (this is my k8s network, hence the 80. You get me.).

You can then apply the .. thing:

```
[root@edd ~]# kubectl apply -f calico.yaml
[root@edd ~]# kubectl get pods -n kube-system
NAME                                       READY   STATUS              RESTARTS   AGE
calico-kube-controllers-58497c65d5-mdksv   0/1     ContainerCreating   0          60s
calico-node-78smp                          0/1     PodInitializing     0          60s
calico-node-pgd28                          0/1     PodInitializing     0          60s
calico-node-wk6vt                          0/1     PodInitializing     0          60s
coredns-78fcd69978-k5sbq                   0/1     ContainerCreating   0          113m
coredns-78fcd69978-xzqhv                   0/1     ContainerCreating   0          113m
etcd-edd                                   1/1     Running             1          114m
kube-apiserver-edd                         1/1     Running             1          114m
kube-controller-manager-edd                1/1     Running             1          114m
kube-proxy-82xhn                           1/1     Running             0          74m
kube-proxy-8t4kg                           1/1     Running             0          113m
kube-proxy-dz6hp                           1/1     Running             0          74m
kube-scheduler-edd                         1/1     Running             1          114m
```

As you can see, the status has changed to say that it's initializing. Container creating is because I have already attempted to build some containers and failed due to this networking.

FTR I had an issue following Calicos documentation because of my `kubectl init --pod-network-cidr=xxxx` command. I *think* calico attempts to use 192.168.0.0/16 by default, however I wanted to run it on a custom CIDR. The [Project Calico Quickstart](https://docs.projectcalico.org/getting-started/kubernetes/quickstart) documentation seems to infer that it's possible to set the CIDR on init however this is false; you must edit the `calico.yaml` file and set your CIDR in order for it to work.

I hope this makes sense and/or helps somebody. Shout out to [Go Linux Cloud](https://www.golinuxcloud.com/calico-kubernetes/) for actually solving this and posting it.

In the time it took me to write these docs, I have green across the board:

```
[root@edd ~]# kubectl get pods -n kube-system
NAME                                       READY   STATUS    RESTARTS   AGE
calico-kube-controllers-58497c65d5-mdksv   1/1     Running   0          4m7s
calico-node-78smp                          1/1     Running   0          4m7s
calico-node-pgd28                          1/1     Running   0          4m7s
calico-node-wk6vt                          1/1     Running   0          4m7s
coredns-78fcd69978-k5sbq                   1/1     Running   0          117m
coredns-78fcd69978-xzqhv                   1/1     Running   0          117m
etcd-edd                                   1/1     Running   1          117m
kube-apiserver-edd                         1/1     Running   1          117m
kube-controller-manager-edd                1/1     Running   1          117m
kube-proxy-82xhn                           1/1     Running   0          77m
kube-proxy-8t4kg                           1/1     Running   0          117m
kube-proxy-dz6hp                           1/1     Running   0          77m
kube-scheduler-edd                         1/1     Running   1          117m
```

Moving on.


### Installing calicoctl

*Intentionally left blank, due to laziness*

## Assign Roles to Workers

This is one of those answers "because the internet told me". I dont know how to get the available labels or anything like that.

```
[root@edd ~]# kubectl label node ed node-role.kubernetes.io/worker=worker
node/ed labeled
[root@edd ~]# kubectl label node eddie node-role.kubernetes.io/worker=worker
node/eddie labeled
[root@edd ~]# docker get^C
[root@edd ~]# kubectl get nodes
NAME    STATUS   ROLES                  AGE    VERSION
ed      Ready    worker                 85m    v1.22.1
edd     Ready    control-plane,master   124m   v1.22.1
eddie   Ready    worker                 85m    v1.22.1
```