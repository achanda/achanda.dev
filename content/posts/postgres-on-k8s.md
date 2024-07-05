+++
title = 'On Postgres, Kubernetes and the fungibility of workload'
date = 2024-07-05T11:56:52-05:00
draft = false
+++

Into to postgres and k8s
- constraints: vanilla postgres, k8s, single writer multiple readers
- not talking about taking over a current operation (read or write) for fungibility

define workload:
- writes (primary)
- reads
  - consistent reads (primary)
  - eventually consistent reads (replica)

count of case 2 >> count of case 1

define state of system as function of writes

define fungibility, why is it important to k8s

vanilla postgres:
- replica is fungible
- primary is fungible in pool of replicas
