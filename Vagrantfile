#-------------------------------------------------------------------------
# Cluster setup configuration
#-------------------------------------------------------------------------
master_nodes            = 1
worker_nodes            = 1
cluster_endpoint_name   = "my-kubernetes-cluster"
use_loadbalancer        = false

#-------------------------------------------------------------------------
# Network addressing configuration
#-------------------------------------------------------------------------
keepalived_virtual_ip   = "192.168.56.100"
load_balancer_network   = "192.168.56.1"
master_nodes_network    = "192.168.56.2"
worker_nodes_network    = "192.168.56.3"
master_nodes_addresses  = []

#-------------------------------------------------------------------------
# Creates the master_nodes_addresses array to use into the HAProxy
# configuration if LB is enabled
#-------------------------------------------------------------------------
(1..master_nodes).each do |node|
    master_nodes_addresses.append("#{master_nodes_network}#{node}")
end

#-------------------------------------------------------------------------
# Generate Kubernetes authorization token for cluster joining
#-------------------------------------------------------------------------
require 'securerandom'
join_token = SecureRandom.alphanumeric(6).downcase + "." + SecureRandom.alphanumeric(16).downcase

#-------------------------------------------------------------------------
# Script configuration for components setup like Keepalived, HAProxy 
# and Kubernetes Master/Worker nodes
#-------------------------------------------------------------------------
$haproxy_config_script = <<SCRIPT
cat <<-EOF | sudo tee /etc/haproxy/haproxy.cfg
# /etc/haproxy/haproxy.cfg
#---------------------------------------------------------------------
# Global settings
#---------------------------------------------------------------------
global
    log /dev/log local0
    log /dev/log local1 notice
    daemon

#---------------------------------------------------------------------
# common defaults that all the 'listen' and 'backend' sections will
# use if not designated in their block
#---------------------------------------------------------------------
defaults
    mode                    http
    log                     global
    option                  httplog
    option                  dontlognull
    option http-server-close
    option forwardfor       except 127.0.0.0/8
    option                  redispatch
    retries                 1
    timeout http-request    10s
    timeout queue           20s
    timeout connect         5s
    timeout client          20s
    timeout server          20s
    timeout http-keep-alive 10s
    timeout check           10s

#---------------------------------------------------------------------
# apiserver frontend which proxys to the control plane nodes
#---------------------------------------------------------------------
frontend apiserver
    bind *:6443
    mode tcp
    option tcplog
    default_backend apiserver

#---------------------------------------------------------------------
# round robin balancing for apiserver
#---------------------------------------------------------------------
backend apiserver
    option httpchk GET /healthz
    http-check expect status 200
    mode tcp
    option ssl-hello-chk
    balance     roundrobin
#       server node-1 192.168.50.11:6443 check
EOF
SCRIPT

$keepalived_conf_script = <<SCRIPT
cat <<-EOF | sudo tee /etc/keepalived/keepalived.conf
! /etc/keepalived/keepalived.conf
! Configuration File for keepalived
global_defs {
    router_id LVS_DEVEL
}
vrrp_script check_apiserver {
  script "/etc/keepalived/check_apiserver.sh"
  interval 3
  weight -2
  fall 10
  rise 2
}

vrrp_instance VI_1 {
    state BACKUP
    interface enp0s8
    virtual_router_id 51
    priority 100
    authentication {
        auth_type PASS
        auth_pass 100
    }
    virtual_ipaddress {
        #{keepalived_virtual_ip}/24
    }
    track_script {
        check_apiserver
    }
}
EOF
SCRIPT

$check_apiserver_keepalived_script = <<SCRIPT
cat <<-EOF | sudo tee /etc/keepalived/check_apiserver.sh
#!/bin/sh

errorExit() {
    echo "*** \\$*" 1>&2
    exit 1
}

curl --silent --max-time 2 --insecure https://localhost:6443/ -o /dev/null || errorExit "Error GET https://localhost:6443/"
if ip addr | grep -q #{keepalived_virtual_ip}; then
    curl --silent --max-time 2 --insecure https://#{keepalived_virtual_ip}:6443/ -o /dev/null || errorExit "Error GET https://#{keepalived_virtual_ip}:6443/"
fi
EOF
SCRIPT

$deploy_weavenetwork_script = <<-'SCRIPT'
export KUBECONFIG=/home/vagrant/.kube/config
kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
SCRIPT



#-------------------------------------------------------------------------
# VMs configuration
#-------------------------------------------------------------------------
Vagrant.configure("2") do |config|

    # LoadBalancer VMs configuration
    if use_loadbalancer == true
        (1..2).each do |i|
            config.vm.define "lb-#{i}" do |lb|
                lb.vm.box = "ubuntu/focal64"
                lb.vm.hostname = "lb-kubernetes-#{i}.local"
                lb.vm.network "private_network", ip: "#{load_balancer_network}#{i}",
                    hostname: true,
                    virtualbox_intnet: true,
                    virtualbox_intnet: "kubenetwork"

                lb.vm.provider "virtualbox" do |v|
                    v.name = "kubernetes-load-balancer-#{i}"
                    v.memory = 512
                    v.cpus = 1
                end
                lb.vm.provision :shell, inline: "sudo apt-get update && sudo apt-get install haproxy keepalived -y"
                lb.vm.provision :shell, inline: $haproxy_config_script
                lb.vm.provision :shell, inline: $keepalived_conf_script
                lb.vm.provision :shell, inline: $check_apiserver_keepalived_script
                lb.vm.provision :shell, inline: "sudo chmod +x /etc/keepalived/check_apiserver.sh"
                master_nodes_addresses.each do |addr|
                    $register_master_node_script = <<-SCRIPT
                    cat <<EOF | sudo tee -a /etc/haproxy/haproxy.cfg
                    server node-#{i} #{addr}:6443 check
EOF
                    SCRIPT
                    lb.vm.provision :shell, inline: $register_master_node_script
                end
                lb.vm.provision :shell, inline: "sudo systemctl restart haproxy keepalived"
            end
        end
    end

    # Master nodes VMs configuration    
    (1..master_nodes).each do |i|    
        config.vm.box = "ubuntu/focal64"
        config.vm.define "master-#{i}" do |master|            
            master.vm.hostname = "kube-master-node-#{i}.local"
            master.vm.provider "virtualbox" do |v|
                v.name = "kubernetes-master-#{i}"
                v.memory = 2048
                v.cpus = 2
            end            
            master.vm.network "private_network", ip: "#{master_nodes_network}#{i}",
                hostname: true,
                virtualbox_intnet: true,
                virtualbox_intnet: "kubenetwork"
            current_master_node_ip = "#{master_nodes_network}#{i}"
            master.vm.provision :shell, path: "kubernetes-provisioner.sh"
            case use_loadbalancer
            when true
                master.vm.provision :shell, inline: "echo #{keepalived_virtual_ip}    #{cluster_endpoint_name} | sudo tee -a /etc/hosts"
                case i
                when 1
                    master.vm.provision :shell, inline: "sudo kubeadm init --token #{join_token} --control-plane-endpoint #{cluster_endpoint_name} --upload-certs --apiserver-cert-extra-sans=#{keepalived_virtual_ip} --apiserver-advertise-address=#{current_master_node_ip}"
                    master.vm.provision :shell, inline: "mkdir /home/vagrant/.kube && sudo cp /etc/kubernetes/admin.conf /home/vagrant/.kube/config && chown vagrant:vagrant /home/vagrant/.kube/config"
                    master.vm.provision :shell, inline: "cp /home/vagrant/.kube/config /vagrant/kubeconfig"
                    master.vm.provision :shell, inline: $deploy_weavenetwork_script
                else
                    master.vm.provision :shell, inline: "sudo kubeadm join --token #{join_token} --control-plane --discovery-token-unsafe-skip-ca-verification --apiserver-advertise-address=#{current_master_node_ip} #{cluster_endpoint_name}"
                end
            else
                master.vm.provision :shell, inline: "echo #{current_master_node_ip}    #{cluster_endpoint_name} | sudo tee -a /etc/hosts"
                case i
                when 1
                    master.vm.provision :shell, inline: "sudo kubeadm init --token #{join_token} --control-plane-endpoint #{cluster_endpoint_name} --upload-certs --apiserver-cert-extra-sans=#{current_master_node_ip} --apiserver-advertise-address=#{current_master_node_ip}"
                    master.vm.provision :shell, inline: "mkdir /home/vagrant/.kube && sudo cp /etc/kubernetes/admin.conf /home/vagrant/.kube/config && chown vagrant:vagrant /home/vagrant/.kube/config"
                    master.vm.provision :shell, inline: "cp /home/vagrant/.kube/config /vagrant/kubeconfig"
                    master.vm.provision :shell, inline: $deploy_weavenetwork_script
                else
                    master.vm.provision :shell, inline: "sudo kubeadm join --token #{join_token} --control-plane --discovery-token-unsafe-skip-ca-verification --apiserver-advertise-address=#{current_master_node_ip} #{cluster_endpoint_name}"
                end
            end
        end
    end

    # Worker nodes VMs configuration
    (1..worker_nodes).each do |i|    
        config.vm.box = "ubuntu/focal64"
        config.vm.define "worker-#{i}" do |worker|            
            worker.vm.hostname = "kube-worker-node-#{i}.local"            
            worker.vm.provider "virtualbox" do |v|
                v.name = "kubernetes-worker-#{i}"
                v.memory = 2048
                v.cpus = 2
            end            
            worker.vm.network "private_network", ip: "#{worker_nodes_network}#{i}",
                hostname: true,
                virtualbox_intnet: true,
                virtualbox_intnet: "kubenetwork"
            current_worker_node_ip = "#{worker_nodes_network}#{i}"
            case use_loadbalancer
            when true
                worker.vm.provision :shell, inline: "echo #{keepalived_virtual_ip}    #{cluster_endpoint_name} | sudo tee -a /etc/hosts"
            else
                worker.vm.provision :shell, inline: "echo #{master_nodes_network}1    #{cluster_endpoint_name} | sudo tee -a /etc/hosts"
            end
            worker.vm.provision :shell, path: "kubernetes-provisioner.sh"
            worker.vm.provision :shell, inline: "sudo kubeadm join --token #{join_token} --apiserver-advertise-address=#{current_worker_node_ip} --discovery-token-unsafe-skip-ca-verification #{cluster_endpoint_name}:6443"
        end
    end
end