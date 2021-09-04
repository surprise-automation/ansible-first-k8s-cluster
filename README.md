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

What you need to do is literally told to you. We've already created the "regular" user (icekube), so:

```
root> cp /etc/kubernetes/admin.conf /home/icekube/.kube/
root> chown icekube:icekube /home/icekube/.kube/admin.conf
root> su icekube
icekube> kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
```

This will apply the Calcio pod network. I dont fully understand the *why* right this very moment, but it was recommended to me by a friend as a starting point. It definitely sounds quite flexible, and doesn't seem to require extra work to use.

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