# rds-workload

RDS core workload.

## Design principles

 - A workload that resembles Webscale in terms of number of resources and topology (pod to svc ratio, svc to endpoint, etc.)
 - A mix of best effort and guaranteed QoS pods (dpdk pods) (one entire NUMA node for guaranted QoS dpdk pods)
 - E-W data traffic in the form of liveness probes
 - LB type services provided by MetalLB for ingress traffic, with 50% of the namespaces having services of type LoadBalancer (will also try to make this ratio configurable)
 - Consolidate on a single workload more akin to cluster-density than to node-density, and leverage it both for control plane and worker node stressing if needed
 - A canned workload that could be reused among QE and perf/scale testing, able to run in small scales (even a local ovn-k8s kind.sh cluster if needed for CI purposes) and adjustable in size, number of cores per dpdk pods, etc.
 - Don't redo what multi-bench is doing, as we ideally have one tool for kernel networking performance measurment.

## Object replicas

| Iterations / nodes / namespaces   | 1   | 120   |
| --------------------------------- | --- | ----- |
| configmaps                        | 30  | 3600  |
| deployments_best_effort           | 25  | 3000  |
| deployments_dpdk                  | 10  | 1200  |
| endpoints (25 x service)          | 500 | 60000 |
| networkPolicy                     | 3   | 360   |
| namespaces                        | 1   | 120   |
| pods_best_effort (2 x deployment) | 50  | 6000  |
| pods_dpdk (1 x deployment)        | 10  | 1200  |
| route                             | 2   | 240   |
| services                          | 20  | 2400  |
| secrets                           | 42  | 5040  |

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
| DPDK_CORES     | Number of 4-core dpdk pods (should fill all the isolated cores of one NUMA node) | 6 for CPT lab / Dell R760   |
|                |                                                                                  | 3 for Scalelab / Dell FC640 |