In Kubernetes, scheduling refers to making sure that [Pods](https://kubernetes.io/docs/concepts/workloads/pods/) are matched to [Nodes](https://kubernetes.io/docs/concepts/architecture/nodes/) so that the [kubelet](https://kubernetes.io/docs/reference/generated/kubelet) can run them. Preemption is the process of terminating Pods with lower [Priority](https://kubernetes.io/docs/concepts/scheduling-eviction/pod-priority-preemption/#pod-priority) so that Pods with higher Priority can schedule on Nodes. Eviction is the process of terminating one or more Pods on Nodes.

# Scheduling
- [Kubernetes Scheduler](https://kubernetes.io/docs/concepts/scheduling-eviction/kube-scheduler/)
- [Assigning Pods to Nodes](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/)
- [Pod Overhead](https://kubernetes.io/docs/concepts/scheduling-eviction/pod-overhead/)
- [Pod Topology Spread Constraints](https://kubernetes.io/docs/concepts/scheduling-eviction/topology-spread-constraints/)
- [Taints and Tolerations](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/)
- [Scheduling Framework](https://kubernetes.io/docs/concepts/scheduling-eviction/scheduling-framework/)
- [Dynamic Resource Allocation](https://kubernetes.io/docs/concepts/scheduling-eviction/dynamic-resource-allocation/)
- [Scheduler Performance Tuning](https://kubernetes.io/docs/concepts/scheduling-eviction/scheduler-perf-tuning/)
- [Resource Bin Packing for Extended Resources](https://kubernetes.io/docs/concepts/scheduling-eviction/resource-bin-packing/)
- [Pod Scheduling Readiness](https://kubernetes.io/docs/concepts/scheduling-eviction/pod-scheduling-readiness/)
- [Descheduler](https://github.com/kubernetes-sigs/descheduler#descheduler-for-kubernetes)


There are two supported ways to configure the filtering and scoring behavior of the scheduler:

1. [Scheduling Policies](https://kubernetes.io/docs/reference/scheduling/policies/) allow you to configure _Predicates_ for filtering and _Priorities_ for scoring.
2. [Scheduling Profiles](https://kubernetes.io/docs/reference/scheduling/config/#profiles) allow you to configure Plugins that implement different scheduling stages, including: `QueueSort`, `Filter`, `Score`, `Bind`, `Reserve`, `Permit`, and others. You can also configure the kube-scheduler to run different profiles.


## Pod Scheduling Readiness


# Pod Disruption
[Pod disruption](https://kubernetes.io/docs/concepts/workloads/pods/disruptions/) is the process by which Pods on Nodes are terminated either voluntarily or involuntarily.

Voluntary disruptions are started intentionally by application owners or cluster administrators. Involuntary disruptions are unintentional and can be triggered by unavoidable issues like Nodes running out of resources, or by accidental deletions.

- [Pod Priority and Preemption](https://kubernetes.io/docs/concepts/scheduling-eviction/pod-priority-preemption/)
- [Node-pressure Eviction](https://kubernetes.io/docs/concepts/scheduling-eviction/node-pressure-eviction/)
- [API-initiated Eviction](https://kubernetes.io/docs/concepts/scheduling-eviction/api-eviction/)

