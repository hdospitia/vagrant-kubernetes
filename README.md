# Installation

Pending

It will put the admin kubeconfig file on the working  
directory to connect to the new Kubernetes cluster.  
You can choose one of the following options to connect  
locally to the deployed cluster:  

## Option A
Set the configuration using the KUBECONFIG envvar and  
kubeconfig the file into the current directory.  

```console
export KUBECONFIG=$(pwd)/kubeconfig
```

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

Visit the [Kubernetes Cheat Sheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/#bash) for tips!
