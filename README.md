# k3s single node test environment

## Description

This playbook will set up k3s single node for the testing.
- install k3s
- install Ceph Rook as CSI
- install MetalLB with layer2 mode
- install AWX

## Tested environment

CentOS Stream 9

## Reference

- [smallab-k8s-pve-guide](https://github.com/ehlesp/smallab-k8s-pve-guide/tree/main)

## MetalLB test yaml

Use [this file](./k8s_test_yaml/loadbalancer.yaml)

```text
# kubectl apply -f loadbalancer.yaml 

# kubectl get svc nginx 
NAME    TYPE           CLUSTER-IP     EXTERNAL-IP      PORT(S)        AGE
nginx   LoadBalancer   10.43.122.34   192.168.126.51   80:30842/TCP   3m7s

$ curl -s -o /dev/null -w "%{http_code}\n" 192.168.126.51 

200
```