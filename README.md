# rds-workload

| Iterations / nodes / namespaces   | 1   | 120   |
| --------------------------------- | --- | ----- |
| configmaps                        | 30  | 3600  |
| deployments_best_effort           | 25  | 3000  |
| deployments_dpdk                  | 15  | 1800  |
| endpoints (25 x service)          | 500 | 60000 |
| networkPolicy                     | 3   | 360   |
| namespaces                        | 1   | 120   |
| pods_best_effort (2 x deployment) | 50  | 6000  |
| pods_dpdk (1 x deployment)        | 1   | 1800  |
| route                             | 2   | 240   |
| services                          | 20  | 2400  |
| secrets                           | 42  | 5040  |

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
   - DPDK_PODS: Number of 4-core dpdk pods (should fill one NUMA node)

I.e.: In this example NUMA node0 CPUs are even numbers:
![](./img/mba-on.png)