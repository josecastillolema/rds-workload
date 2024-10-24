# rds-workload

RDS core workload.

## Design principles

 - A workload that resembles Webscale in terms of number of resources (at a 2x scale, twice the objects) and topology (pod to service ratio, service to endpoint, etc.).
 - A mix of best effort and guaranteed QoS pods (DPDK pods). One entire NUMA node for guaranted QoS DPDK pods.
 - E-W data traffic in the form of readiness probes.
 - LB type services provided by MetalLB for ingress traffic.
 - Consolidate on a single workload more akin to cluster-density than to node-density, and leverage it both for control plane and worker node stressing if needed.
 - A canned workload that can be reused among QE and perf/scale testing, able to run in small scales (even a local ovn-k8s kind.sh cluster if needed for CI purposes) and adjustable in size, number of cores per DPDK pod, etc.
 - Don't redo what multi-bench is doing, as we ideally have one tool for kernel networking performance measurement.
 - After the workload creation, apply churn to the workload during 30 min, by randomly deleting 10% of the corresponding namespaces.

## Object replicas

| Iterations / nodes / namespaces   | 1    | 120                                 |
| --------------------------------- | ---- | ----------------------------------- |
| configmaps                        | 30   | 3600                                |
| deployments_best_effort           | 25   | 3000                                |
| deployments_dpdk                  | 2    | 240 (assuming 24 worker-dpdk nodes) |
| endpoints (210x service)          | 4200 | 504000                              |
| endpoints lb (90 x service)       | 90   | 10800                               |
| networkPolicy                     | 3    | 360                                 |
| namespaces                        | 1    | 120                                 |
| pods_best_effort (2 x deployment) | 50   | 6000                                |
| pods_dpdk (1 x deployment)        | 2    | 240 (assuming 24 worker-dpdk nodes) |
| route                             | 2    | 240                                 |
| services                          | 20   | 2400                                |
| services (lb)                     | 1    | 120                                 |
| secrets                           | 42   | 5040                                |

## DPDK pods

DPDK QoS guaranteed pods are emulated using stress-ng consuming 100% CPU on each core.

Each DPDK pod has:
 - ((Total worker CPUs-reserved cores(usually 4))/DPDK_CORES)/10 cores (enough to fill one NUMA node with 10 pods)

     The number of cores per DPDK pod needs to be even, otherwise we will get a SMT Alignment Error.
 - 1 GB memory
 - 16 GB hugepages
 - 1 OVN interface
 - 2 SRIOV interfaces

The ideia is that DPDK pods should consume the entire first NUMA node. I.e.: In this example NUMA node0 CPUs are even numbers and cores 0 and 32 are reserved:
     ![](./img/dpdk_pods.png)

## Input parameters

### General

| Parameter               | Description                                                         | Default value     |
| ----------------------- | ------------------------------------------------------------------- | ----------------- |
| BURST                   | Burst                                                               | 20                |
| CHURN                   | Enable churning                                                     | true              |
| CHURN_CYCLES            | Churn cycles to execute                                             | -                 |
| CHURN_DELAY             | Time to wait between each churn                                     | 2m                |
| CHURN_DELETION_STRATEGY | Churn deletion strategy to use                                      | default           |
| CHURN_DURATION          | Churn duration                                                      | 30m               |
| CHURN_PERCENT           | Percentage of job iterations that kube-burner will churn each round | 10                |
| ES_SERVER               | Elastic Search endpoint                                             | -                 |
| GC_METRICS              | Collect metrics during garbage collection                           | true              |
| GC                      | Garbage collect created namespaces                                  | true              |
| JOB_ITERATIONS          | Number of iterations                                                | number of workers |
| LOCAL_INDEXING          | Enable local indexing                                               | false             |
| POD_READY_THRESHOLD     | Pod ready timeout threshold                                         | 30s               |
| QPS                     | Queries per Second                                                  | 20                |
| SVC_LATENCY             | Enable service latency measurement                                  | true              |


### Specific to the workload

| Parameter      | Description                                                                                      | Default value               |
| -------------- | ------------------------------------------------------------------------------------------------ | --------------------------- |
| INGRESS_DOMAIN | For E-W traffic                                                                                  | -                           |
| DPDK_CORES     | Number of cores assigned for each DPDK pod (should fill all the isolated cores of one NUMA node) | 6 for CPT lab / Dell R760   |
|                |                                                                                                  | 2 for Scalelab / Dell FC640 |

## Pre-requisites

An OpenShift cluster with:
 - A **PerformanceProfile** with isolated and reserved cores, 1G hugepages and and `topologyPolicy=single-numa-node`. Hugepages should be allocated in the first NUMA node (the one that would be used by DPDK deployments):
     ```yaml
      hugepages:
      defaultHugepagesSize: 1G
      pages:
      - count: 160
        node: 0
        size: 1G
      - count: 6
        node: 1
        size: 1G
     ```
 - **MetalLB operator** limiting speaker pods to specific nodes (aprox. 10%, 12 in the case of 120 node iterations with the corresponding ***worker-metallb*** label):
     ```yaml
     apiVersion: metallb.io/v1beta1
     kind: MetalLB
     metadata:
       name: metallb
       namespace: metallb-system
     spec:
       nodeSelector:
         node-role.kubernetes.io/worker-metallb: ""
       speakerTolerations:
       - key: "Example"
         operator: "Exists"
         effect: "NoExecute"
     ```
 - **SRIOV operator** with a corresponding SriovNetworkNodePolicy
 - Some nodes (i.e.: 25% of them) with the ***worker-dpdk*** label to host the DPDK pods, i.e.:
     ```
     $ kubectl label node worker1 node-role.kubernetes.io/worker-dpdk=
     ```

## Run

Example run with 1 iteration:
```
$ METRICS=metrics-report.yml ALERTS=alerts.yml DPDK_CORES=2 PERF_PROFILE=customcnf kube-burner-ocp init -c rds.yml --gc=true --gc-metrics=true --es-server=https://user:pwd@esserver --es-index=kube-burner --service-latency=true --churn=true --iterations 1
```
