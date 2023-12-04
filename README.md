# k3s single node test environment

- [k3s single node test environment](#k3s-single-node-test-environment)
  - [Description](#description)
  - [Tested environment](#tested-environment)
  - [MetalLB test](#metallb-test)
  - [Rook test](#rook-test)
  - [AWX](#awx)
  - [Gitea](#gitea)
  - [Private container registry](#private-container-registry)
  - [Uninstall k3s](#uninstall-k3s)

## Description

This playbook will set up k3s single node for the testing.
- install k3s
- install Ceph Rook as CSI[1]
- install MetalLB with layer2 mode[2]
- install AWX

<br>[1] To use Rook, add an additional block device(/dev/vdb etc) for Ceph BlueStore
<br>[2] To use MetalLB, disable service lb(klipper lb). You can disable servicelb in group_vars

## Tested environment

CentOS Stream 9 running under KVM
- vCPUs 6
- Memory 16GB
- /dev/vda(60GB), /dev/vdb(60GB)
- single vNIC
- SELinux: Permissve

<br>***Run the playbook with root user***

## MetalLB test

Use [this file](./k8s_test_yaml/loadbalancer.yaml)

```text
kubectl apply -f loadbalancer.yaml 
```

```text
kubectl get svc nginx 

NAME    TYPE           CLUSTER-IP     EXTERNAL-IP      PORT(S)        AGE
nginx   LoadBalancer   10.43.122.34   192.168.126.51   80:30842/TCP   3m7s
```

```text
curl -s -o /dev/null -w "%{http_code}\n" http://192.168.126.51 

200
```

## Rook test

```text
kubectl get -n rook-ceph pod

NAME                                            READY   STATUS      RESTARTS   AGE
rook-ceph-operator-b89ccd545-fntdz              1/1     Running     0          36m
rook-ceph-mon-a-64fbd4f87d-d7hp7                1/1     Running     0          33m
csi-rbdplugin-wb55g                             2/2     Running     0          32m
csi-cephfsplugin-xwl9z                          2/2     Running     0          32m
rook-ceph-mgr-a-5957bb6774-slsg8                1/1     Running     0          32m
csi-rbdplugin-provisioner-799c7dc8b-86gzc       5/5     Running     0          32m
csi-cephfsplugin-provisioner-6cf65b9555-8ffx2   5/5     Running     0          32m
rook-ceph-osd-prepare-lab04-k3-skjrj            0/1     Completed   0          32m
rook-ceph-osd-0-7f9bb8d8d6-5htgs                1/1     Running     0          31m
rook-ceph-tools-757999d6c7-wmhq6                1/1     Running     0          31m
rook-ceph-mds-myfs-a-b7488c544-qjnq8            1/1     Running     0          30m
rook-ceph-exporter-lab04-k3-6f4c485d87-7tq9d    1/1     Running     0          30m
rook-ceph-mds-myfs-b-544d87b98f-tfvp9           1/1     Running     0          30m
```

```text
kubectl get -n rook-ceph storageclasses.storage.k8s.io 

NAME                   PROVISIONER                  RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
local-path (default)   rancher.io/local-path        Delete          WaitForFirstConsumer   false                  37m
rook-ceph-block        rook-ceph.rbd.csi.ceph.com   Delete          Immediate              true                   30m
```

```text
kubectl exec -n rook-ceph rook-ceph-tools-757999d6c7-wmhq6 -- ceph status

  cluster:
    id:     071550c8-e649-4633-ac46-c4218b38ded0
    health: HEALTH_OK
 
  services:
    mon: 1 daemons, quorum a (age 35m)
    mgr: a(active, since 34m)
    mds: 1/1 daemons up, 1 hot standby
    osd: 1 osds: 1 up (since 34m), 1 in (since 34m)
 
  data:
    volumes: 1/1 healthy
    pools:   4 pools, 112 pgs
    objects: 22 objects, 465 KiB
    usage:   27 MiB used, 60 GiB / 60 GiB avail
    pgs:     112 active+clean
 
  io:
    client:   1.2 KiB/s rd, 2 op/s rd, 0 op/s wr
```

```text
kubectl exec -n rook-ceph rook-ceph-tools-757999d6c7-wmhq6 -- ceph osd df

ID  CLASS  WEIGHT   REWEIGHT  SIZE    RAW USE  DATA     OMAP  META    AVAIL   %USE  VAR   PGS  STATUS
 0    hdd  0.05859   1.00000  60 GiB   27 MiB  888 KiB   0 B  26 MiB  60 GiB  0.04  1.00  112      up
                       TOTAL  60 GiB   27 MiB  888 KiB   0 B  26 MiB  60 GiB  0.04                   
MIN/MAX VAR: 1.00/1.00  STDDEV: 0
```

<br>Try [this](https://github.com/rook/rook/tree/master/deploy/examples)
```text
kubectl apply -f mysql.yaml 
kubectl apply -f wordpress.yaml 
```

```
kubectl get pv

NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                    STORAGECLASS      REASON   AGE
pvc-1140df78-1c63-4ac8-a09b-d991ae4cc9b9   20Gi       RWO            Delete           Bound    default/mysql-pv-claim   rook-ceph-block            2m44s
pvc-a211b752-f858-4acc-ab8a-a730709829be   20Gi       RWO            Delete           Bound    default/wp-pv-claim      rook-ceph-block            2m39s
```
```text
kubectl get pvc

NAME             STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS      AGE
mysql-pv-claim   Bound    pvc-1140df78-1c63-4ac8-a09b-d991ae4cc9b9   20Gi       RWO            rook-ceph-block   2m46s
wp-pv-claim      Bound    pvc-a211b752-f858-4acc-ab8a-a730709829be   20Gi       RWO            rook-ceph-block   2m42s
```

## AWX

[Installation Guide](https://ansible.readthedocs.io/projects/awx-operator/en/latest/installation/basic-install.html)

<br>Check the port number
```
# kubectl get service -n awx 
NAME                                              TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
awx-operator-controller-manager-metrics-service   ClusterIP   10.43.214.109   <none>        8443/TCP       33m
awx-demo-postgres-13                              ClusterIP   None            <none>        5432/TCP       7m1s
awx-demo-service                                  NodePort    10.43.230.70    <none>        80:31470/TCP   5m59s
```

<br>Get the credentials of a user admin
```text
kubectl get secrets -n awx awx-demo-admin-password -o jsonpath="{.data.password}" | base64 --decode ; echo
```
Open the browser and access to the `http//[K3 host ip]:31470`

## Gitea

Open the browser, access to `http://[k3 host ip]:30000`
```
# kubectl get po -l app=gitea
NAME                     READY   STATUS    RESTARTS   AGE
gitea-744f954f44-4xsj9   1/1     Running   0          114s

# kubectl get svc -l app=gitea
NAME         TYPE       CLUSTER-IP    EXTERNAL-IP   PORT(S)          AGE
gitea-http   NodePort   10.43.89.28   <none>        3000:30000/TCP   2m14s
gitea-ssh    NodePort   10.43.100.5   <none>        22:30001/TCP     2m14s
```

## Private container registry

Referecnce:
- https://github.com/docker-archive/docker-registry/tree/master/contrib/nginx
 
## Uninstall k3s

```shell
k3s-killall.sh 
```

```shell
k3s-uninstall.sh
```

<br>If you have installed Rook, do the followings.
```text
rm -rf /var/lib/rook
```

<br>Wipefs the block deivce for Rook
```shell
wipefs -af /dev/vdb 
```
