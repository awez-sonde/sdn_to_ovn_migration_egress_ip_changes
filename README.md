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
