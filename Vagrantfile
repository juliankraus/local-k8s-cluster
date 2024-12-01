# -*- mode: ruby -*-
# vi: set ft=ruby :

# Kubernetes cluster configuration
K8S_VERSION = '1.30.5'
K8S_CONTROL_PLANE_BOOTSTRAP_TOKEN = 'abcdef.0123456789abcdef'
K8S_KUBECONFIG_CA_CERT_PATH = './ansible/roles/role_install_k8s_control_plane_debian/files/ca.crt'
K8S_KUBECONFIG_CLUSTER_NAME = 'k8s-vagrant-local'
K8S_KUBECONFIG_SERVER_ADDRESS = '192.168.100.10'
K8S_KUBECONFIG_SERVER_PORT = 6443
K8S_KUBECONFIG_CREDENTIALS_NAME = 'vagrant'
K8S_KUBECONFIG_CREDENTIALS_CLIENT_CERT_PATH = './ansible/roles/role_install_k8s_control_plane_debian/files/vagrant.crt'
K8S_KUBECONFIG_CREDENTIALS_CLIENT_KEY_PATH = './ansible/roles/role_install_k8s_control_plane_debian/files/vagrant.key'
K8S_KUBECONFIG_CREDENTIALS_USERNAME = 'vagrant'
K8S_KUBECONFIG_CONTEXT_NAME = 'k8s-vagrant-local'

# Convert a string to a number greater than 40.000
def string_to_number(str)
    # Calculate hash value based on ASCII values of string characters
    hash_value = str.chars.map(&:ord).sum

    (hash_value % 15001) + 40000
end

Vagrant.configure("2") do |config|
    # global configs
    config.vm.box_check_update = false
    # rsync workspace into vm
    config.vm.synced_folder ".", "/vagrant", type: "rsync", rsync__args: ["--archive", "--delete", "-z", "--copy-links", "--executability"]
    
    vm_configs = [
        { name: "k8s-control-plane", box: "bento/ubuntu-24.04", box_version: "202404.26.0", ip: "192.168.100.10", mac: "52:54:00:12:34:56" },
        { name: "k8s-worker-node-1", box: "bento/ubuntu-24.04", box_version: "202404.26.0", ip: "192.168.100.20", mac: "78:8D:0B:3F:30:89"},
        { name: "k8s-worker-node-2", box: "bento/ubuntu-24.04", box_version: "202404.26.0", ip: "192.168.100.30", mac: "B6:9B:C8:7C:3E:45" }
    ]

    vm_configs.each do |vm|
        # dynamically create vagrant vms
        config.vm.define vm[:name] do |node|
            node.vm.box = vm[:box]
            node.vm.box_version = vm[:box_version]
            
            node.vm.hostname = vm[:name]
            
            # Configure chrony, user, ssh
            # Install basic Kubernetes components
            node.vm.provision "ansible_local" do |ansible|
                ansible.playbook = "ansible/playbook_vagrant_provisioning.yaml"
                # Install the latest ansible version from apt
                ansible.install = true
                ansible.install_mode = "default"
                ansible.extra_vars = {
                    k8s_node_eth0_ip_address: vm[:ip],
                    k8s_node_eth0_default_gateway: "192.168.100.1",
                    k8s_version: K8S_VERSION
                }
            end
            
            # Setup a k8s control plane or worker node
            if vm[:name].include?("control-plane")
               node.vm.provider "qemu" do |qe|
                   # configure a vmnet network on eth0 and port-forwarding on eth1, forward Kubernetes API server port to localhost                   
                   qe.extra_qemu_args = ["-device", "virtio-net-pci,netdev=net0,mac=" + vm[:mac], "-netdev", "socket,id=net0,fd=3", "-device", "virtio-net-pci,netdev=net1", "-netdev", "user,id=net1,hostfwd=tcp::" + string_to_number(vm[:name]).to_s + "-:22", "-device", "virtio-net-pci,netdev=net2", "-netdev", "user,id=net2,hostfwd=tcp::6443-:6443"]
               end
               node.vm.provision "ansible_local" do |ansible|
                   ansible.playbook = "ansible/playbook_k8s_control_plane.yaml"
                   ansible.extra_vars = {
                       k8s_control_plane_bootstrap_token: K8S_CONTROL_PLANE_BOOTSTRAP_TOKEN,
                       k8s_version: K8S_VERSION
                   }
               end               
                # set current kubeconfig context to vagrant cluster after setup
                node.trigger.after :up do |trigger|
                    trigger.info = "Create kubeconfig entry for vagrant cluster"
                    trigger.run = {
                        inline: "bash -c 'kubectl config set-cluster #{K8S_KUBECONFIG_CLUSTER_NAME} --certificate-authority=#{K8S_KUBECONFIG_CA_CERT_PATH} --embed-certs=true --server=https://#{K8S_KUBECONFIG_SERVER_ADDRESS}:#{K8S_KUBECONFIG_SERVER_PORT} && 
                                          kubectl config set-credentials #{K8S_KUBECONFIG_CREDENTIALS_NAME} --client-certificate=#{K8S_KUBECONFIG_CREDENTIALS_CLIENT_CERT_PATH} --client-key=#{K8S_KUBECONFIG_CREDENTIALS_CLIENT_KEY_PATH} --embed-certs=true --username=#{K8S_KUBECONFIG_CREDENTIALS_USERNAME} &&
                                          kubectl config set-context #{K8S_KUBECONFIG_CONTEXT_NAME} --cluster=#{K8S_KUBECONFIG_CLUSTER_NAME} --user=#{K8S_KUBECONFIG_CREDENTIALS_NAME} && 
                                          kubectl config use-context #{K8S_KUBECONFIG_CONTEXT_NAME}'"
                    }
                    trigger.on_error = :continue
                end                
                # delete vagrant cluster from kubeconfig context after destroying the cluster
                node.trigger.after :destroy do |trigger|
                    trigger.info = "Delete kubeconfig entry for vagrant cluster"
                    trigger.run = {
                        inline: "bash -c 'kubectl config unset current-context &&
                                          kubectl config delete-context #{K8S_KUBECONFIG_CONTEXT_NAME} &&
                                          kubectl config delete-user #{K8S_KUBECONFIG_CREDENTIALS_NAME} &&
                                          kubectl config delete-cluster #{K8S_KUBECONFIG_CLUSTER_NAME}'"
                    }
                    trigger.on_error = :continue
                end
            elsif vm[:name].include?("worker-node")
               node.vm.provider "qemu" do |qe|
                   # configure a vmnet network on eth0 and port-forwarding on eth1                   
                   qe.extra_qemu_args = ["-device", "virtio-net-pci,netdev=net0,mac=" + vm[:mac], "-netdev", "socket,id=net0,fd=3", "-device", "virtio-net-pci,netdev=net1", "-netdev", "user,id=net1,hostfwd=tcp::" + string_to_number(vm[:name]).to_s + "-:22"]
               end            
               node.vm.provision "ansible_local" do |ansible|
                   ansible.playbook = "ansible/playbook_k8s_worker_node.yaml"
                   ansible.extra_vars = {
                       k8s_control_plane_host: "192.168.100.10",
                       k8s_control_plane_port: 6443,
                       k8s_control_plane_bootstrap_token: K8S_CONTROL_PLANE_BOOTSTRAP_TOKEN
                   }
               end
            end 
            
            # qemu provider config
            node.vm.provider "qemu" do |qe|
                qe.qemu_dir = "/usr/local/share/qemu"
                qe.arch = "x86_64"
                qe.machine = "q35,accel=hvf"
                qe.cpu = "max"
                qe.net_device = nil
                qe.ssh_port = string_to_number(vm[:name])
            end
        end
    end    
end
