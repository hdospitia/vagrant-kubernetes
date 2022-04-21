# Installation

Pending...

It will put the admin kubeconfig file on the working
directory to connect to the new Kubernetes cluster.
You can choose one of the following options to connect
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
