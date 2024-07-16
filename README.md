# rds-workload

| Iterations / nodes / namespaces | 120   | 1   |
| ------------------------------- | ----- | --- |
| deployments                     | 3000  | 25  |
| pods (2 x deployment)           | 6000  | 50  |
| services                        | 2400  | 20  |
| endpoints (25 x service)        | 60000 | 500 |
| configmaps                      | 3600  | 30  |
| secrets                         | 5040  | 42  |
| route                           | 240   | 2   |
| networkPolicy                   | 360   | 3   |