# Brief

This Vagrant configuration let you deploy a basic,
local Kubernetes cluster with the Weave-Net CNI plugin.
Also it is possible to launch a highly available load
balancer using HAProxy and KeepAlived, but it is
disabled by default since it requires more horse power.

***Note:*** Please don't hate me, but I tested it on Linux
environment, and works perfectly. Windows can be more RAM
and CPU greedy, so may be provisioning VMs never ends.

# Deployment

Take a quick look on the variables at the top of the
Vagrantfile. Those allows you to select how many master
and worker nodes you want to deploy. Additionally, you
can decide between deploy a HAProxy load balancer or not.
It is disabled by default. Be familiarized with the
values set and be sure that those matches with your
capacity and intentions.

Keep in mind that more nodes or even enabling the load
balancer will demand more resource power (Two VMs will be
launched to run HAProxy and KeepAlived). It's recommended
to plan and validate if you have enough resources in your
machine before to deploy or modify defaults.

Once ready to deploy, hit:

```console
vagrant up
```

You must use a mean to translate names to IPs (DNS) such as the
hosts file or a DNS server. Take care of the IP used by the
master-1 or the load balancer virtual IP, due you need to put
it into your /etc/hosts file with your "cluster_endpoint_name",
which is used on the generated kubeconfig file.

For instance, if you did enable the load balancer by
```use_loadbalancer = true```, just use the value of the variable
"keepalived_virtual_ip" and put it into your /etc/hosts file,
translating to the value of "cluster_endpoint_name" variable, like
this:
```console
echo "192.168.56.100   my-kubernetes-cluster" | sudo tee -a /etc/hosts
```

If you use defaults, just hit:
```console
echo "192.168.56.21   my-kubernetes-cluster" | sudo tee -a /etc/hosts
```

At the end of the deployment, Vagrant put the admin kubeconfig
file on the working directory to connect to the new Kubernetes
cluster. You can choose one of the following options to connect
locally to the deployed cluster:

## Option A
Set the configuration using the KUBECONFIG envvar and
kubeconfig the file into the current directory. Make
sure you are still inside of this repo when launch the
command:

```console
export KUBECONFIG=$(pwd)/kubeconfig
```

### Note:
This leverages the [Vagrant capability](https://www.vagrantup.com/docs/synced-folders#synced-folders)
of mount the Vagrantfile parent folder inside of
the VM on /vagrant path. On the Vagrantfile, I copy
the generated kubeconfig to the /vagrant/ folder,
which makes it available for you into the repo directory.

## Option B
Move the kubeconfig file to the $HOME/.kube folder and
set the KUBECONFIG envvar to use it. This lets you to
have the kubeconfig file into a recommended path without
impact your current $HOME/.kube/config file, that may be
present.

```console
[[ ! -d ~/.kube ]] && mkdir ~/kube
cp ./kubeconfig ~/.kube/vagrant-cluster.config
export KUBECONFIG=~/.kube/vagrant-cluster.config
```

Get the nodes status once you configured properly your
environment:

```console
kubectl get nodes
```

Visit the [Kubernetes Cheat Sheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/#bash)
for tips!
