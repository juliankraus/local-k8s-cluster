# local-k8s-cluster
Setup a three-node Kubernetes cluster on an Intel Mac using QEMU, socket_vmnet, Vagrant &amp; Ansible.

## Prerequisites
This setup requires a local subnet 192.168.100.0/32 on the host. I used [socket_vmnet](https://github.com/lima-vm/socket_vmnet) for this. A detailed description of how to set up socket_vmnet can be found [in this Medium article](https://medium.com/@kraus-julian/93a979078274).

If you have the kubectl CLI tool available on your host, vagrant also sets the context of kubectl to use the local cluster.

## Configure Kubernetes Version
In the Vagrantfile set the **K8S_VERSION** variable to a Kubernetes release you'd like to install.

## Start the cluster
Execute the following in the directory containing the Vagrantfile.
```
vagrant up
```

## Destroy the cluster
Execute the following in the directory containing the Vagrantfile.
```
vagrant destroy -f
```

