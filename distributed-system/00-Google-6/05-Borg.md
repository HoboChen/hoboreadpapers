Borg is  

Cluster manager 

Across a number of cluster with each up to ~10k machines 

 

Process-level performance isolation 

 

Offer declarative job specification language, name service intergration, real-time job monitoring, and tools to analyze and simulate system behavior 

 

User perspective 

 

Heterogenous workload, long-running(disurnal usage pattern), batch jobs 

 

GFS, CFS, BitTable, MegaStore all runs on Borg 

 

`prod` and `non-prod`, 

 

Prod-cpu: allocate: 70%, represent: 60% 

Prod-mem: allocate:55%, represent: 85% 

 

2.2 clusters and cells 

 

Machines in a cell belong to a single cluster, datacenter-scale network fabric 

A cluster lives inside a single datacenter building 

A collection of buildings make a site 

 

Median cell size: 10k; heterogeneous 

 

2.3 jobs and tasks 

 

Jobs -> name, owner, tasks 

Jobs can have constraints 

 

Tasks -> a set of linux processes running in a container 

Vast majority of the borg workloads does not run inside VM 

 

Statically linked, structured as packages of binaries and data files, whose installation is orchestrated by Borg 

 

Operate on Borg jobs by RPC, Borg job or automation system (e.g. monitoring) 

 

First SIGTERM (80% gracefully), then SIGKILL 

 

2.4 alloc 

 

a reserved set of resources on a machine 

 

2.5 priority, quota, and admission control 

 

preemption cascades could occur 

 

Quota is used to decide which jobs to admit for scheduling 

 

Prod quota is limited to the actual resources available in the cell 

 

Quota allocation is handled outside of Borg, and is intimately tied to our physical capacity planning 

 

2.6 naming and monitoring 

 

3 borg architecture 

 

3.1  

 

Why only most? 

 

Single elected master pre cell serves both as the Paxos leader and the state mutator 

And acquires a chubby lock, so can be discovered 

 

Failing over ~10s; but no stale in the cluster makes ~ 1min, (why not lock based and piggy back like chubby?) 