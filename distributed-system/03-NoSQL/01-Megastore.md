## [Megastore: Providing Scalable, Highly Available Storage for Interactive Services]()

Megastore blends the scalability of a NoSQL datastore with the convenience of a traditional RDBMS in a novel way,
and provides both strong consistency guarantees and high availability.

### Introduction

Meeting storage demands of services like email, collaborative documents and social networking is challenging due to a number of conflicting requirements:

- Scalability; a service can be built rapidly using MySQL as its datastore, but scaling the service to millions of users requires a complete redesign of its storage infrasture
- Ease of usage; services must compete for users
- Low latency; responsive
- Consistency; services should provide the user with a *consistent view* of data
- Avaliablity; the service must be resilient to faults ranging from the failure of individual disks, machines, routers, and even entire datacenters

These requirements are in conflict.