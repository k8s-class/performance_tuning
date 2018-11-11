kube-apiserver: Kubernete's REST API entry point that processes operations on Kubernetes objects, i.e. Pods, Deployments, Stateful Sets, Persistent Volume Claims, Secrets, etc. An operation mutates (create / update / delete) or reads a spec describing the REST API object(s)
Etcd: A highly available key-value store for kube-apiserver
kube-controller-manager: Runs control loops that manage objects from kube-apiserver and perform actions to make sure these objects maintain the states described by their specs
kube-scheduler: Gets pending Pods from kube-apiserver, assigns a minion to the Pod on which it should run, and writes the assignments back to API server. kube-scheduler assigns minions based on available resources, QoS, data locality and other policies described in its driving algorithm
kubelet: A Kubernetes worker that runs on each minion. It watches Pods via kube-apiserver and looks for Pods that are assigned to itself. It then syncs these Pods if possible. The procedure of Syncing Pods requires resource provisioning (i.e. mount volume), talking with container runtime to manage Pod life cycle (i.e. pull images, run containers, check container health, delete containers and garbage collect containers)
kube-proxy: A network proxy that reflects Service (defined in Kubernetes REST API) that runs on each node. Watches Service and Endpoint objects from kube-apiserver and modifies the underlying kernel iptable for routing and redirection.

----------------------------------------------------------------------------------
kube-apiserver --max-requests-inflight(Throttle API requests) --target-ram-mb(Control memory consumption)
----------------------------------------------------------------------------------
--max-requests-inflight

This flag will limit the number of API calls that will be processed in parallel, which is a great control point of kube-apiserver memory consumption. The API server can be very CPU intensive when processing a lot of requests in parallel. With the latest Kubernetes release, they provide more fine-grained API throttling mechanisms with "--max-requests-inflight" and "--max-mutating-requests-inflight"

--target-ram-mb

This value is used for kube-apiserver to guess the size of the cluster and to configure the deserialize cache size and watch cache [9] sizes inside the API server.

https://kubernetes.io/docs/reference/command-line-tools-reference/kube-apiserver/
--------------------------------------------------------------------------------
kube-controller-manager
--------------------------------------------------------------------------------
Control level of parallelism

--concurrent-deployment-syncs
 --concurrent-endpoint-syncs
 --concurrent-gc-syncs
 --concurrent-namespace-syncs
 --concurrent-replicaset-syncs
 --concurrent-resource-quota-syncs
 --concurrent-service-syncs
 --concurrent-serviceaccount-token-syncs
 --concurrent-rc-syncs


Kube-controller-manager has a set of flags that can provide fine-grained controls of parallelism. Increasing parallelism means Kubernetes will be more agile when updating specs, but also allows the controller manager to consume more CPU and memory.
In general, increase settings for components you use more intensively. For larger clusters, feel free to increase default values as long as you are OK with its memory usage. For smaller clusters, if you are tight on memory, you can lower the settings. Kubernetes default values can be found in the kube-controller-manager documentation.


-------------------------------------------------------------------------------
 Control memory consumption

--replication-controller-lookup-cache-size
--replicaset-lookup-cache-size
--daemonset-lookup-cache-size

 This set of flags is not documented but still available for use. Increasing look up cache size can increase sync speed of corresponding controllers, but will increase memory consumption for controller manager. In practice, we set memory request for controller manager container to be greater than the sum of these 3 values. Note that after Kubernetes 1.6, replication controller, replica set and daemonset no longer require a lookup cache so these flags are not needed.
 The default values (4GB for Replication Controller, 4GB for Replica Set and 1GB for Daemon Set) works fine for even large workloads. You can tune these values down for smaller clusters to save memory.
 Set the container's memory request to slightly greater than the sum of these values.
--------------------------------------------------------------------------------

Throttle API query rate

--kube-api-burst
 --kube-api-qps

These 2 flags set normal and burst rate that controller manager can talk to kube-apiserver. We increase these values with larger Applatix cluster configs

Default values work pretty well (20 for QPS and 30 for burst). Increase these values for large, production clusters

--------------------------------------------------------------------------------
kube-scheduler
-------------------------------------------------------------------------------

Throttle API query rate

--kube-api-burst
--kube-api-qps

These 2 flags sets normal and burst rate that kube-scheduler can talk to kube-apiserver, as kube-scheduler polls "need-to-schedule" Pods from api-server, and writes scheduling decisions. [11] [12] kube-scheduler's memory consumption can increase noticeably when there is a burst of pod creation happening inside the cluste
Set QPS and Burst to roughly 20% and 30% of max inflight requests for API server is a good assumption to make.
-------------------------------------------------------------------------------
kube-proxy
-------------------------------------------------------------------------------

Throttle API query rate

--kube-api-burst
--kube-api-qps

These 2 flags sets normal and burst rate that kube-scheduler can talk to kube-apiserver. kube-proxy mainly use kube-apiserver to watch for changes in Service and Endpoint objects

The default values are fine.
-------------------------------------------------------------------------------
etcd
-------------------------------------------------------------------------------
--snapshot-count

Etcd memory consumption and disk usage are directly affected by `--snapshot-count`, and it is likely to have memory burst when there are a lot of Pod churns cluster-wide. See more information about etcd tuning

We reduced the default values significantly but keep it enough for our usage. The default value is unnecessarily large for our use case, and can easily cause etcd to be OOM killed.



-------------------------------------------------------------------------------
kubelet
-------------------------------------------------------------------------------
Throttle API query rate

--kube-api-burst
--kube-api-qps

These 2 flags sets normal and burst rate that kubelet can talk to kube-apiserver. As each node will have a limited number of pods, default values work pretty good for us in our stress tests

Control event generation rate

--event-burst
--event-qps

These 2 flags controls rate kubelet creates events. More events means more workload for master node to process, and you need to tune this value if your have some micro service analyzing (especially caching) Kubernetes event streams, as these services can possibly get OOMed when kubelets are generating too many events globally

Throttle container registry query rate

--registry-burst
--registry-qps

Rate control when talking to container registry. In our daily development

Default values are fine. Increase the value if your application is sensitive to the delay of pulling container images.


Protect host

--max-open-files
--max-pods
--kube-reserved
--system-reserved

These 4 flags helps kubelet protect the host.

Set `--max-pods` to prevent too many small containers from overloading the kubelet for a node. Generally, limit it to 80 or so Pods per node (roughly about 300 containers) per m3.2xlarge node.

The `--kube-reserved` `--system-reserved` are also useful flags to reserve resources for system and kubernetes components (such as kernel, docker, kubelet, and other Kubernetes Daemon Sets). kube-scheduler will take this into consideration when scheduling pods
