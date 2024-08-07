# rds-workload

RDS core workload.

## Object replicas

| Iterations / nodes / namespaces               | 1   | 120   |
| --------------------------------------------- | --- | ----- |
| configmaps                                    | 30  | 3600  |
| deployments_best_effort                       | 25  | 3000  |
| deployments_dpdk (CTP lab / R760)             | 15  | 1800  |
| deployments_dpdk (scalelab / FC640)           | 7   | 840   |
| endpoints (25 x service)                      | 500 | 60000 |
| networkPolicy                                 | 3   | 360   |
| namespaces                                    | 1   | 120   |
| pods_best_effort (2 x deployment)             | 50  | 6000  |
| pods_dpdk (1 x deployment) (cpt LAB / R760)   | 15  | 1800  |
| pods_dpdk (1 x deployment) (scalelab / FC640) | 7   | 840   |
| route                                         | 2   | 240   |
| services                                      | 20  | 2400  |
| secrets                                       | 42  | 5040  |

## DPDK pods

DPDK QoS guaranteed pods are emulated using stress-ng consuming 100% CPU on each core.

Each DPDK pod has:
 - 4 cores
 - 1 GB memory
 - 1 GB hugepage
 - 1 OVN interface
 - 2 SRIOV interfaces

The ideia is that they consume an entire NUMA node. I.e.: In this example NUMA node0 CPUs are even numbers and cores 0 and 64 are reserved:
     ![](./img/dpdk_pods.png)

## Input parameters

 - General
   - BURST
   - CHURN
   - CHURN_CYCLES
   - CHURN_DELAY
   - CHURN_DELETION_STRATEGY
   - CHURN_DURATION
   - CHURN_PERCENT
   - ES_SERVER
   - GC_METRICS
   - GC
   - JOB_ITERATIONS
   - LOCAL_INDEXING
   - POD_READY_THRESHOLD
   - QPS
   - SVC_LATENCY

 - Specific to the workload
   - INGRESS_DOMAIN: For E-W traffic
   - DPDK_PODS: Number of 4-core dpdk pods (should fill all the isolated cores of one NUMA node)