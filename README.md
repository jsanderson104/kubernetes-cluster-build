
<h1> Ansible Role to build an On-Prem Kubernetes Cluster on CentOS 9 </h1>

To EXECUTE the Ansible Role, ```bash build.sh``` <br>
To MANIPULATE how the Ansible role executs, edit the ```build.yml``` file. <br><br>

* It's IMPERATIVE that tasks that are interacting with the Kubernetes cluster via ```kubectl``` NEVER run with ```become=true``` or they WILL FAIL. <br>
* In this role, I only configure the automation user with kubernetes credentials.

---
This Ansible Role will build a Kubernetes Cluster with <br>
* 1 x Master Node <br>
* 3 x Worker Nodes<br>

What is Included with the Cluster Build from this Ansible Role:
* As stated above, a 4-node cluster with 3 Workers
* The "automation" user Kubernetes credentials pre-configured on the "master" node
* Flannel CNI working... using the POD NET CIDR variable in the role.
* MetalLB load-balancer for giving our services a way to be exposed to the external LAN
* A "Web App Demo" service that is exposed to the LAN to prove that MetalLB load-balancer is working.
* A pod running on each worker node, I called "Ping Pods", that can be used to test connectivity between worker nodes using the POD NET.
* Joined to FreeIPA domain if variable(s) are defined in build.yml
---


---
<b> What is NOT included:</b>
* The VM's obviously and the platform to run them on.
* Common-Sense... hopefully you have some.
* Support for this Role
* IPA servers to join --> If you don't have a FreeIPA domain to join just set the joinipa variable to false.
  

---
I am using Oracle Virtualbox on my Windows workstation to run all of the VM's talked about here. <br>
All of the Kubernetes nodes VM's NIC are set to ```Promiscuous mode``` and they are configured as a ```Bridged Adapter``` so that the VM's can fully interact with my home LAN. <br>
Each Future Kubernetes node should have:
* 100GB disk
* One NIC on same subnet as all other nodes and ideally be in the same subnet as your LAN.
* Running CentOS 9
* At least 4GB RAM
* At least 2 CPU cores, preferably 4.

<b>I have a total of 7 VM's running:</b>
1. ansible "server"
2. ipa server 1
3. ipa server 2
4. kubernetes master node
5. kubernetes worker1
6. kubernetes worker2
7. kubernetes worker3

<b>Lab Setup:</b>
1. Each VM is in the hosts file in this role via a template file
2. Each VM has a user with full sudo privileges. In my case the user is called "automation"
3. The "automation" user should be able to password-less ssh to all nodes listed above.
4. The role provides a hosts file template. Modify that accordingly. The entries in hosts file should match the ansible_host variable in your inventory file.
---



<b>Ansible Collections needed:</b>
* Look at the ```ansible.cfg``` file to figure out where to put the collections (-p option)
```
[defaults]
remote_user=automation
roles_path = ../roles:/usr/share/ansible/roles
collections_path = ../collections:/usr/share/ansible/collections
```

```
ansible-galaxy collection install ansible.posix -p ./collections/
```

``` 
ansible-galaxy collection install community.general -p ./collections/
```

---
The Components of the Kubernetes Cluster:
* We're setting the Cluster up with the following:
  * <b> Containerd </b>              --> Our actual CRI that's responsible for executing/running the containers
  * <b> Kube-Proxy </b>              --> The broker of incoming/outgoing operations to the Control-plane API
  * <b> Kubelet </b>                 --> <b>Forgot.... where's my book again. It interacts with the CRI for some reason???</b>
  * <b> Flannel CNI </b>             --> This is what enables PODs to talk to other PODs on different nodes. We supply the CIDR for this in the variables.
  * <b> MetalLB LoadBalancer </b>    --> This is what uses the LAN IP range and brokers traffic into the services of the Kubernetes cluster.
  * <b> Control-Plane </b>           --> This is the brains of the operation -aka- "master" nodes. 
  * <b> CoreDNS </b>                 --> This is what allows a pod to talk to another pod on the same or a different host by container name via POD NET
  * <b> HeadLamp </b>                --> This isn't necessary for the Kubernetes Cluster to function at all. This is just a handy addon that gives us a Web Interface to manage/monitor the Cluster with.
---

***
  Let's talk about FIPS mode. The ONLY reason I developed the FIPS stuff in this role was because my current deployment of a FreeIPA cluster was built with FIPS 140.2 enabled.
  The Kubernetes cluster nodes (CENTOS9) default crypto-policy attempts to use a non-FIPS approved cipher when trying to join the FreeIPA servers and automatically fails.
  Therefore, we have to put the cluster in FIPS:AD-SUPPORT  ``` sudo update-crypto-policies --set FIPS:AD-SUPPORT ``` crypto policy so the ipa join will be forced at the kernel level to use a stronger cipher to talk to the server. I have since re-deployed my FreeIPA cluster without FIPS to make things simpler for testing and learning purposes. The functionality in the role still works.
  WARNING: Some Java applets inside containers will fail if FIPS is running. The Java process in the container will have an error mentioning libcrypto.so and MD (aka MD5).
           Guacamole Frontend container is an example of this problem. It runs a openjdk applet. 
***

# How to install the driver for NFS CSI</h4>
This NFS CSI "driver" is needed to allow deplyoments/pods,etc to dynamically provision NFS storage. <br>
We can request storage from the storageclass with just a PVC since it the driver dynamically makes the PV for us when we make the storage claim.<br>
<a href=https://github.com/kubernetes-csi/csi-driver-nfs/blob/master/docs/install-csi-driver-v4.13.4.md> Instructions for Installing NFS Driver </a>

To install it, run this command on the master node.
```
curl -skSL https://raw.githubusercontent.com/kubernetes-csi/csi-driver-nfs/v4.13.4/deploy/install-driver.sh | bash -s v4.13.4 --
```
In my home lab, I'm using a second disk on my Ansible VM and exporting it with NFS to the subnet where my nodes live.
  


# HEADLAMP NOTES:
If you're wondering how to Login to HeadLamp, you can generate a login token using this command: 
```
kubectl create token headlamp -n kube-system
```




  
