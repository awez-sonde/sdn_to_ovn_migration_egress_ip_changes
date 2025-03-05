# Egress IP changes for SDN to OVN migration

## Working environment

Cluster version

```
[root@ampere-mtsnow-altra-10 ~]# oc get clusterversion
NAME      VERSION   AVAILABLE   PROGRESSING   SINCE   STATUS
version   4.14.48   True        False         14h     Cluster version is 4.14.48
```

Nodes , netnamespaces and hostsubnets

```
[root@ampere-mtsnow-altra-10 ~]# oc get nodes
NAME                                STATUS   ROLES                  AGE   VERSION
cluster1-ctlplane-0.awezlab.local   Ready    control-plane,master   15h   v1.27.16+03a907c
cluster1-ctlplane-1.awezlab.local   Ready    control-plane,master   15h   v1.27.16+03a907c
cluster1-ctlplane-2.awezlab.local   Ready    control-plane,master   15h   v1.27.16+03a907c
cluster1-worker-0.awezlab.local     Ready    worker                 14h   v1.27.16+03a907c
cluster1-worker-1.awezlab.local     Ready    worker                 14h   v1.27.16+03a907c
cluster1-worker-2.awezlab.local     Ready    worker                 14h   v1.27.16+03a907c
cluster1-worker-3.awezlab.local     Ready    worker                 14h   v1.27.16+03a907c
cluster1-worker-4.awezlab.local     Ready    worker                 14h   v1.27.16+03a907c
cluster1-worker-5.awezlab.local     Ready    worker                 14h   v1.27.16+03a907c


```

No `netnamespaces` has been assigned `egress IP`

```

[root@ampere-mtsnow-altra-10 ~]# oc get netnamespaces | grep -vE "openshift|kube" 
NAME                                               NETID      EGRESS IPS
default                                            0          
kcli-infra                                         3884936    
test                                               16155714   

```

No `hostsubnet` has been assigned any `egress IP`

```
[root@ampere-mtsnow-altra-10 ~]# oc get hostsubnet
NAME                                HOST                                HOST IP           SUBNET          EGRESS CIDRS   EGRESS IPS
cluster1-ctlplane-0.awezlab.local   cluster1-ctlplane-0.awezlab.local   192.168.122.144   10.134.0.0/23                  
cluster1-ctlplane-1.awezlab.local   cluster1-ctlplane-1.awezlab.local   192.168.122.19    10.132.0.0/23                  
cluster1-ctlplane-2.awezlab.local   cluster1-ctlplane-2.awezlab.local   192.168.122.71    10.133.0.0/23                  
cluster1-worker-0.awezlab.local     cluster1-worker-0.awezlab.local     192.168.122.160   10.132.4.0/23                  
cluster1-worker-1.awezlab.local     cluster1-worker-1.awezlab.local     192.168.122.227   10.133.2.0/23                  
cluster1-worker-2.awezlab.local     cluster1-worker-2.awezlab.local     192.168.122.228   10.132.2.0/23                  
cluster1-worker-3.awezlab.local     cluster1-worker-3.awezlab.local     192.168.122.195   10.135.0.0/23                  
cluster1-worker-4.awezlab.local     cluster1-worker-4.awezlab.local     192.168.122.223   10.134.2.0/23                  
cluster1-worker-5.awezlab.local     cluster1-worker-5.awezlab.local     192.168.122.198   10.135.2.0/23                  

```

Verifying the network type

```
[root@ampere-mtsnow-altra-10 ~]# oc get network cluster -o yaml
apiVersion: config.openshift.io/v1
kind: Network
metadata:
  creationTimestamp: "2025-03-03T19:20:50Z"
  generation: 2
  name: cluster
  resourceVersion: "3857"
  uid: 11b5e176-6d05-4046-8536-3706798b4d9e
spec:
  clusterNetwork:
  - cidr: 10.132.0.0/14
    hostPrefix: 23
  externalIP:
    policy: {}
  networkType: OpenShiftSDN
  serviceNetwork:
  - 172.30.0.0/16
status:
  clusterNetwork:
  - cidr: 10.132.0.0/14
    hostPrefix: 23
  clusterNetworkMTU: 1450
  networkType: OpenShiftSDN
  serviceNetwork:
  - 172.30.0.0/16


```

## Creating environemnt to test Egress IP working

I have created a test VM outside openshift cluster.This will will help us identify if the Egress IP is working fine

```
[cloud-user@testvm ~]$ systemctl status httpd
● httpd.service - The Apache HTTP Server
     Loaded: loaded (/usr/lib/systemd/system/httpd.service; disabled; preset: disabled)
     Active: active (running) since Tue 2025-03-04 06:16:35 EST; 7min ago
       Docs: man:httpd.service(8)
   Main PID: 11201 (httpd)
     Status: "Total requests: 9; Idle/Busy workers 100/0;Requests/sec: 0.0205; Bytes served/sec:  43KB/sec"
      Tasks: 177 (limit: 2471)
     Memory: 13.8M
        CPU: 261ms
     CGroup: /system.slice/httpd.service
             ├─11201 /usr/sbin/httpd -DFOREGROUND
             ├─11202 /usr/sbin/httpd -DFOREGROUND
             ├─11203 /usr/sbin/httpd -DFOREGROUND
             ├─11204 /usr/sbin/httpd -DFOREGROUND
             └─11205 /usr/sbin/httpd -DFOREGROUND

Mar 04 06:16:35 testvm systemd[1]: Starting The Apache HTTP Server...
Mar 04 06:16:35 testvm httpd[11201]: AH00558: httpd: Could not reliably determine the server's fully qualified domain name, using 192.168.122.177. Set the 'ServerName' dir>
Mar 04 06:16:35 testvm httpd[11201]: Server configured, listening on: port 80
Mar 04 06:16:35 testvm systemd[1]: Started The Apache HTTP Server.


[cloud-user@testvm ~]$ ip -4 a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    altname enp3s0
    inet 192.168.122.177/24 brd 192.168.122.255 scope global dynamic noprefixroute eth0
       valid_lft 2921sec preferred_lft 2921sec
```

Performing a curl from a pod within `test` namespace to this VM IP i.e `192.168.122.177` and then taking tcpdumps on VM to identify the incoming IP 

![image](https://github.com/user-attachments/assets/dd4da418-dd40-4dca-b74d-3363b8618a9b)

The incoming IP is 192.168.122.228 i.e the node IP where the pod resides.

We can confirm this below.

```
[root@ampere-mtsnow-altra-10 ~]# oc get po -o wide
NAME                          READY   STATUS    RESTARTS   AGE   IP            NODE                              NOMINATED NODE   READINESS GATES
httpd-hello-f485fb875-2jw4n   1/1     Running   0          25m   10.132.2.10   cluster1-worker-2.awezlab.local   <none>           <none>


[root@ampere-mtsnow-altra-10 ~]# oc get node -o wide | grep -i cluster1-worker-2.awezlab.local
cluster1-worker-2.awezlab.local     Ready    worker                 15h   v1.27.16+03a907c   192.168.122.228   <none>        Red Hat Enterprise Linux CoreOS 414.92.202502111902-0 (Plow)   5.14.0-284.104.1.el9_2.aarch64   cri-o://1.27.8-13.rhaos4.14.gitaf2f916.el9
```


## Assigning `egressIP` to `netnamespace` and `hostsubnet`

In this example, we will use Egress IPs ` 192.168.122.4 ` till `192.168.122.6` with dynamic assignment and `192.168.122.7` till `192.168.122.9` for manual assignment

Dynamic assignment ---> 192.168.122.4 - 192.168.122.6
Manual Assignment ---> 192.168.122.7 - 192.168.122.9



Assigning Egress IP to the namespace

```
[root@ampere-mtsnow-altra-10 ~]# oc patch netnamespace test --type=merge -p  '{
    "egressIPs": [
      "192.168.122.4"
    ]                
  }'
netnamespace.network.openshift.io/test patched

[root@ampere-mtsnow-altra-10 ~]# oc get netnamespace | grep -i test
test                                               16155714   ["192.168.122.4"]

```

Assigning Egress CIDR to worker-0 with dynamic assignment

```
[root@ampere-mtsnow-altra-10 ~]# oc patch hostsubnet cluster1-worker-0.awezlab.local --type=merge -p   '{
    "egressCIDRs": [
      "192.168.122.0/29"
      ]              
  }'
hostsubnet.network.openshift.io/cluster1-worker-0.awezlab.local patched


[root@ampere-mtsnow-altra-10 ~]# oc get hostsubnet 
NAME                                HOST                                HOST IP           SUBNET          EGRESS CIDRS           EGRESS IPS
cluster1-ctlplane-0.awezlab.local   cluster1-ctlplane-0.awezlab.local   192.168.122.144   10.134.0.0/23                          
cluster1-ctlplane-1.awezlab.local   cluster1-ctlplane-1.awezlab.local   192.168.122.19    10.132.0.0/23                          
cluster1-ctlplane-2.awezlab.local   cluster1-ctlplane-2.awezlab.local   192.168.122.71    10.133.0.0/23                          
cluster1-worker-0.awezlab.local     cluster1-worker-0.awezlab.local     192.168.122.160   10.132.4.0/23   ["192.168.122.0/29"]   ["192.168.122.4"]
cluster1-worker-1.awezlab.local     cluster1-worker-1.awezlab.local     192.168.122.227   10.133.2.0/23                          
cluster1-worker-2.awezlab.local     cluster1-worker-2.awezlab.local     192.168.122.228   10.132.2.0/23                          
cluster1-worker-3.awezlab.local     cluster1-worker-3.awezlab.local     192.168.122.195   10.135.0.0/23                          
cluster1-worker-4.awezlab.local     cluster1-worker-4.awezlab.local     192.168.122.223   10.134.2.0/23                          
cluster1-worker-5.awezlab.local     cluster1-worker-5.awezlab.local     192.168.122.198   10.135.2.0/23
```

Adding a new EgressIP to another project

```
[root@ampere-mtsnow-altra-10 ~]# oc patch netnamespace newtest --type=merge -p  '{
    "egressIPs": [
      "192.168.122.7"
    ]
  }'
netnamespace.network.openshift.io/newtest patched
```
Assigning Egress CIDR to worker-1 with manual assignment

```

[root@ampere-mtsnow-altra-10 ~]#  oc patch hostsubnet cluster1-worker-1.awezlab.local --type=merge -p \
  '{
    "egressIPs": [
      "192.168.122.7"                                                                  
      ]             
  }'                                                                                   
hostsubnet.network.openshift.io/cluster1-worker-1.awezlab.local patched


[root@ampere-mtsnow-altra-10 ~]# oc get hostsubnet
NAME                                HOST                                HOST IP           SUBNET          EGRESS CIDRS           EGRESS IPS
cluster1-ctlplane-0.awezlab.local   cluster1-ctlplane-0.awezlab.local   192.168.122.144   10.134.0.0/23                          
cluster1-ctlplane-1.awezlab.local   cluster1-ctlplane-1.awezlab.local   192.168.122.19    10.132.0.0/23                          
cluster1-ctlplane-2.awezlab.local   cluster1-ctlplane-2.awezlab.local   192.168.122.71    10.133.0.0/23                          
cluster1-worker-0.awezlab.local     cluster1-worker-0.awezlab.local     192.168.122.160   10.132.4.0/23   ["192.168.122.0/29"]   ["192.168.122.4"]
cluster1-worker-1.awezlab.local     cluster1-worker-1.awezlab.local     192.168.122.227   10.133.2.0/23                          ["192.168.122.7"]
cluster1-worker-2.awezlab.local     cluster1-worker-2.awezlab.local     192.168.122.228   10.132.2.0/23                          
cluster1-worker-3.awezlab.local     cluster1-worker-3.awezlab.local     192.168.122.195   10.135.0.0/23                          
cluster1-worker-4.awezlab.local     cluster1-worker-4.awezlab.local     192.168.122.223   10.134.2.0/23                          
cluster1-worker-5.awezlab.local     cluster1-worker-5.awezlab.local     192.168.122.198   10.135.2.0/23       
```


Verifying if the curl from pods are going through Egress Ips 

![image](https://github.com/user-attachments/assets/7a93497f-62a2-454a-8ad4-d9b03cf93a06)


## Performing the migration 

Using https://github.com/awez-sonde/ovnmigration-networkpolicies 


## Post migraton status

Verifying if the migration was successfull

```
[root@ampere-mtsnow-altra-10 ~]# oc get network.config/cluster -o jsonpath='{.status.networkType}{"\n"}'
OVNKubernetes


```

Check the egress Ips

```
[root@ampere-mtsnow-altra-10 ~]# oc get hostsubnet
NAME                                HOST                                HOST IP           SUBNET          EGRESS CIDRS   EGRESS IPS
cluster1-ctlplane-0.awezlab.local   cluster1-ctlplane-0.awezlab.local   192.168.122.144   10.134.0.0/23                  
cluster1-ctlplane-1.awezlab.local   cluster1-ctlplane-1.awezlab.local   192.168.122.19    10.132.0.0/23                  
cluster1-ctlplane-2.awezlab.local   cluster1-ctlplane-2.awezlab.local   192.168.122.71    10.133.0.0/23                  
cluster1-worker-0.awezlab.local     cluster1-worker-0.awezlab.local     192.168.122.160   10.132.4.0/23                  
cluster1-worker-1.awezlab.local     cluster1-worker-1.awezlab.local     192.168.122.227   10.133.2.0/23                  
cluster1-worker-2.awezlab.local     cluster1-worker-2.awezlab.local     192.168.122.228   10.132.2.0/23                  
cluster1-worker-3.awezlab.local     cluster1-worker-3.awezlab.local     192.168.122.195   10.135.0.0/23                  
cluster1-worker-4.awezlab.local     cluster1-worker-4.awezlab.local     192.168.122.223   10.134.2.0/23                  
cluster1-worker-5.awezlab.local     cluster1-worker-5.awezlab.local     192.168.122.198   10.135.2.0/23                  

[root@ampere-mtsnow-altra-10 ~]# oc get netnamespaces | grep -i test
newtest                                            9713796    
test                                               16155714   
```

Both the hostsubnet and netnamespace doesnt seem to have an IP , but lets check egressIP object which is part of OVNKubernetes

```

[root@ampere-mtsnow-altra-10 ~]# oc get egressip
NAME               EGRESSIPS       ASSIGNED NODE                     ASSIGNED EGRESSIPS
egressip-newtest   192.168.122.7   cluster1-worker-1.awezlab.local   192.168.122.7
egressip-test      192.168.122.4   cluster1-worker-1.awezlab.local   192.168.122.4

```

Both egress IP's are automatically created.

Now we check the nodes if the label `k8s.ovn.org/egress-assignable=""` is assigned automatically or not

```
[root@ampere-mtsnow-altra-10 ~]# oc get node --show-labels | grep -i egress
cluster1-worker-0.awezlab.local     Ready    worker                 41h   v1.27.16+03a907c   beta.kubernetes.io/arch=arm64,beta.kubernetes.io/os=linux,k8s.ovn.org/egress-assignable=,kubernetes.io/arch=arm64,kubernetes.io/hostname=cluster1-worker-0.awezlab.local,kubernetes.io/os=linux,node-role.kubernetes.io/worker=,node.openshift.io/os_id=rhcos

cluster1-worker-1.awezlab.local     Ready    worker                 41h   v1.27.16+03a907c   beta.kubernetes.io/arch=arm64,beta.kubernetes.io/os=linux,k8s.ovn.org/egress-assignable=,kubernetes.io/arch=arm64,kubernetes.io/hostname=cluster1-worker-1.awezlab.local,kubernetes.io/os=linux,node-role.kubernetes.io/worker=,node.openshift.io/os_id=rhcos

```

As can be seen, only worker-0 and worker-1 have been labeled automatically since those are the only two host subnets we patched.

Lets identify which IP is assigned to which node

```
[root@ampere-mtsnow-altra-10 ~]# oc get egressip egressip-test -o json | jq .status
{
  "items": [
    {
      "egressIP": "192.168.122.4",
      "node": "cluster1-worker-1.awezlab.local"
    }
  ]
}
[root@ampere-mtsnow-altra-10 ~]# 
[root@ampere-mtsnow-altra-10 ~]# 
[root@ampere-mtsnow-altra-10 ~]# oc get egressip egressip-newtest -o json | jq .status
{
  "items": [
    {
      "egressIP": "192.168.122.7",
      "node": "cluster1-worker-1.awezlab.local"
    }
  ]
}


```

Let's check the high availability. I will shutdown the worker1 so the egress ip should move to worker-0

```

[root@ampere-mtsnow-altra-10 ~]# oc debug node/cluster1-worker-1.awezlab.local
Temporary namespace openshift-debug-srp86 is created for debugging node...
Starting pod/cluster1-worker-1awezlablocal-debug-qkdcv ...
To use host binaries, run `chroot /host`
Pod IP: 192.168.122.227
If you don't see a command prompt, try pressing enter.
sh-4.4# chroot /host
sh-5.1# shutdown now
sh-5.1# 
Removing debug pod ...
Temporary namespace openshift-debug-srp86 was removed.


[root@ampere-mtsnow-altra-10 ~]# oc get nodes
NAME                                STATUS     ROLES                  AGE   VERSION
cluster1-ctlplane-0.awezlab.local   Ready      control-plane,master   42h   v1.27.16+03a907c
cluster1-ctlplane-1.awezlab.local   Ready      control-plane,master   42h   v1.27.16+03a907c
cluster1-ctlplane-2.awezlab.local   Ready      control-plane,master   42h   v1.27.16+03a907c
cluster1-worker-0.awezlab.local     Ready      worker                 42h   v1.27.16+03a907c
cluster1-worker-1.awezlab.local     NotReady   worker                 42h   v1.27.16+03a907c
cluster1-worker-2.awezlab.local     Ready      worker                 42h   v1.27.16+03a907c
cluster1-worker-3.awezlab.local     Ready      worker                 42h   v1.27.16+03a907c
cluster1-worker-4.awezlab.local     Ready      worker                 42h   v1.27.16+03a907c
cluster1-worker-5.awezlab.local     Ready      worker                 42h   v1.27.16+03a907c

```

Now lets check the Egress Ips

```
[root@ampere-mtsnow-altra-10 ~]# oc get egressip egressip-newtest -o json | jq .status
{
  "items": [
    {
      "egressIP": "192.168.122.7",
      "node": "cluster1-worker-0.awezlab.local"
    }
  ]
}
[root@ampere-mtsnow-altra-10 ~]# oc get egressip egressip-test -o json | jq .status
{
  "items": [
    {
      "egressIP": "192.168.122.4",
      "node": "cluster1-worker-0.awezlab.local"
    }
  ]
}


```

Both of the egressIps successfully moved to worker-0 
