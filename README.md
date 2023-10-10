# free5gc_deployment_using_microk8s
I have created this repository just to help the people who are trying to deploy the free5gc network using Kubernetes

Step by step procedure to create free5gc network using Kubernetes

1. Download & install Oracle Virtual Box from the following link : https://www.virtualbox.org/wiki/Downloads

2. Create a VM with following specifications :
   ![image](https://github.com/ssdumbre/free5gc_deployment_using_microk8s/assets/43669080/150e2859-0d30-4e2b-8966-280787550423)

3. Download and install Ubuntu on the above VM (Link to download VM -> https://ubuntu.com/download/desktop: ubuntu-22.04.2-desktop-amd64)
   
4. Clone the repository for gtp5g - 5G compatible GTP kernel module from : https://github.com/free5gc/gtp5g

Compile :

git clone https://github.com/free5gc/gtp5g.git && cd gtp5g
make clean && make

Install kernel module (Install the module to the system and load automatically at boot) :

sudo make install   

5. Install the required packages for user-plane :

sudo apt -y install git gcc g++ cmake autoconf libtool pkg-config libmnl-dev libyaml-dev

6. Install microk8s with the help of following commands:

sudo snap install microk8s --classic
sudo snap list 
newgrp microk8s
sudo usermod -a -G microk8s <userid_using_microk8s>
sudo chown -f -R <userid_using_microk8s> ~/.kube

7. Install the command line tool for microk8s i.e., nothing but kubectl with the help of following commands:

curl -LO https://storage.googleapis.com/kubernetes-release/release/`curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt`/bin/linux/amd64/kubectl
chmod +x kubectl
sudo mv kubectl /usr/local/bin/

8. Install helm (The Package Manager to install free5gc helm chart for the ue , access and core network) with the help of following commands:

curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh

9. Start a single-node cluster and enable Multus-CNI

microk8s disable ha-cluster --force
microk8s enable dns ingress dashboard storage community helm3
microk8s enable multus

10. Add & pull the repositories of free5gc & ueramsim using below Helm Commands :
cd /root/ ; mkdir kubedata
mkdir /root/5gc ; cd /root/5gc
helm repo add towards5gs 'https://raw.githubusercontent.com/Orange-OpenSource/towards5gs-helm/main/repo/'
helm repo update
helm search repo
helm pull towards5gs/free5gc; helm pull towards5gs/ueransim

11. Create a persistent volume : 
cd /root/5gc/
vi pv.yaml
---------
apiVersion: v1
kind: PersistentVolume
metadata:
  name: example-local-pv9
  labels:
    project: free5gc
spec:
  capacity:
    storage: 8Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  local:
    path: /root/kubedata
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - <nodename>

13. Check the physical network interface on Kubernetes node & if the names of network interfaces are different from eth0 and eth1 then we will have to apply the Networks configuration as mentioned on https://github.com/Orange-OpenSource/towards5gs-helm/tree/main/charts/free5gc#networks-configuration

Use ip addr command to check the physical network interface

root@______:~/5gc# ip addr

1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000

    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00

    inet 127.0.0.1/8 scope host lo

       valid_lft forever preferred_lft forever

    inet6 ::1/128 scope host 

       valid_lft forever preferred_lft forever

2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000

    link/ether 08:00:27:b5:20:f4 brd ff:ff:ff:ff:ff:ff

    inet 10.0.2.15/24 brd 10.0.2.255 scope global dynamic noprefixroute enp0s3

       valid_lft 84360sec preferred_lft 84360sec

    inet6 fe80::5b3a:afa8:91b7:6d60/64 scope link noprefixroute 

       valid_lft forever preferred_lft forever

As you can see from the above output of ip addr command the name of the physical network interface is different from eth0 i.e., enp0s3 so I will make the necessary configuration changes into the network.

Enable the promiscous mode [In a network, promiscuous mode allows a network device to intercept and read each network packet that arrives]on this interface by the command: sudo ip link set enp0s3 promisc on

Deploy the Helm chart for free5GC’s core network components as given below :
a. Create a namespace for free5gc:

root@______:~/5gc# microk8s kubectl create ns free5gc
namespace/free5gc created

root@______:~/5gc# microk8s kubectl get ns -n --all
NAME              STATUS   AGE
default           Active   59m
free5gc           Active   5s
ingress           Active   59m
kube-node-lease   Active   59m
kube-public       Active   59m
kube-system       Active   59m

b. Deploy the helm chart using following command: 

root@______:~/5gc# microk8s helm -n free5gc install free5gc-core towards5gs/free5gc --set global.n2network.masterIf=enp0s3,global.n3network.masterIf=enp0s3,global.n4network.masterIf=enp0s3,global.n6network.masterIf=enp0s3,global.n9network.masterIf=enp0s3,global.n6network.subnetIP=10.0.2.0,global.n6network.gatewayIP=10.0.2.1,global.n6network.cidr=24,free5gc-upf.upf.n6if.ipAddress=10.0.2.14,global.n2network.type=macvlan,global.n3network.type=macvlan,global.n4network.type=macvlan,global.n6network.type=macvlan,global.n9network.type=macvlan

Check if the free5gc core network pods are up and running or not by using following command :

root@______:# microk8s kubectl get pods --all-namespaces

NAMESPACE     NAME                                                     READY   STATUS    RESTARTS   AGE
free5gc       free5gc-core-free5gc-amf-amf-7b856846c9-mwmwl            1/1     Running   0          5m19s
free5gc       free5gc-core-free5gc-ausf-ausf-7dd46c9fb7-qlk28          1/1     Running   0          5m20s
free5gc       free5gc-core-free5gc-dbpython-dbpython-b6b587768-m7b67   1/1     Running   0          5m20s
free5gc       free5gc-core-free5gc-nrf-nrf-94c56fb79-qcht9             1/1     Running   0          5m20s
free5gc       free5gc-core-free5gc-nssf-nssf-545f9dc99c-b9rcs          1/1     Running   0          5m20s
free5gc       free5gc-core-free5gc-pcf-pcf-57589b5c66-gtbq4            1/1     Running   0          5m19s
free5gc       free5gc-core-free5gc-smf-smf-7cc7bd6b54-lz2r8            1/1     Running   0          5m20s
free5gc       free5gc-core-free5gc-udm-udm-5d5497c6f4-wjvgr            1/1     Running   0          5m19s
free5gc       free5gc-core-free5gc-udr-udr-ffb6dc48f-2djhw             1/1     Running   0          5m20s
free5gc       free5gc-core-free5gc-upf-upf-795586f9b-dvx9n             1/1     Running   0          5m19s
free5gc       free5gc-core-free5gc-webui-webui-5fbb96469-82lbg         1/1     Running   0          5m20s
free5gc       mongodb-0                                                1/1     Running   0          5m19s
ingress       nginx-ingress-microk8s-controller-mx59w                  1/1     Running   0          64m
kube-system   coredns-7745f9f87f-8qj4j                                 1/1     Running   0          64m
kube-system   dashboard-metrics-scraper-5cb4f4bb9c-4646p               1/1     Running   0          64m
kube-system   hostpath-provisioner-58694c9f4b-7bjqh                    1/1     Running   0          64m
kube-system   kube-multus-ds-cddtb                                     1/1     Running   0          64m
kube-system   kubernetes-dashboard-fc86bcc89-kwgwv                     1/1     Running   0          64m
kube-system   metrics-server-7747f8d66b-qdgzr                          1/1     Running   0          64m























References: 
1. https://github.com/Orange-OpenSource/towards5gs-helm/blob/main/docs/demo/Setup-free5gc-and-test-with-UERANSIM.md#networks-configuration
2. https://github.com/Orange-OpenSource/towards5gs-helm/tree/main/charts/free5gc#networks-configuration
3. https://github.com/devopsjourney23/free5gcdemo/blob/master/README.md
4. https://hackmd.io/@haidinhtuan/ryRuKdEI3

***** Note : I would like to thank the github community to help me create open-source free5gc network with the help of Kubernetes *****
