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

| Iterations / nodes / namespaces   | 1   | 120                                 |
| --------------------------------- | --- | ----------------------------------- |
| configmaps                        | 30  | 3600                                |
| deployments_best_effort           | 25  | 3000                                |
| deployments_dpdk                  | 2   | 240 (assuming 24 worker-dpdk nodes) |
| endpoints (25 x service)          | 500 | 60000                               |
| endpoints lb (2 x service)        | 2   | 240                                 |
| networkPolicy                     | 3   | 360                                 |
| namespaces                        | 1   | 120                                 |
| pods_best_effort (2 x deployment) | 50  | 6000                                |
| pods_dpdk (1 x deployment)        | 2   | 240 (assuming 24 worker-dpdk nodes) |
| route                             | 2   | 240                                 |
| services                          | 20  | 2400                                |
| services (lb)                     | 1   | 120                                 |
| secrets                           | 42  | 5040                                |

## DPDK pods

DPDK QoS guaranteed pods are emulated using stress-ng consuming 100% CPU on each core.

Each DPDK pod has:
 - (worker CPUs-4)/2 cores (enough to fill one NUMA node with 10 pods)
 - 1 GB memory
 - 1 GB hugepage
 - 1 OVN interface
 - 2 SRIOV interfaces

The ideia is that DPDK pods should consume an entire NUMA node. I.e.: In this example NUMA node0 CPUs are even numbers and cores 0 and 64 are reserved:
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

| Parameter      | Description                                                                      | Default value               |
| -------------- | -------------------------------------------------------------------------------- | --------------------------- |
| INGRESS_DOMAIN | For E-W traffic                                                                  | -                           |
| DPDK_CORES     | Number of 4-core DPDK pods (should fill all the isolated cores of one NUMA node) | 6 for CPT lab / Dell R760   |
|                |                                                                                  | 3 for Scalelab / Dell FC640 |

## Pre-requisites

An OpenShift cluster with:
 - A **PerformanceProfile** with isolated and reserved cores, hugepages and and topologyPolicy=single-numa-node
 - **MetalLB operator** limiting speaker pods to specific nodes (aprox. 10%, 12 in the case of 120 node iterations with the corresponding **worker-metallb** label):
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
 - Some nodes (i.e.: 25% of them) with the **worker-dpdk label** to host the dpdk pods, i.e.:
     ```
     $ kubectl label node worker1 node-role.kubernetes.io/worker-dpdk=
     ```
 - Expand the default **node port range**:
     ```
     $ kubectl patch network.config.openshift.io cluster --type=merge -p \
     '{
     "spec":
          { "serviceNodePortRange": "30000-60000" }
     }'
     ```
## Run

Example run with 1 iteration:
```
$ METRICS=metrics-report.yml ALERTS=alerts.yml DPDK_CORES=2 PERF_PROFILE=customcnf kube-burner-ocp init -c rds.yml --gc=true --gc-metrics=true --es-server=https://user:pwd@esserver --es-index=kube-burner --service-latency=true --churn=false --churn=true --iterations 1
```
