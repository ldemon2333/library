The Phase of a Pod is a high-level summary of where the Pod is in its lifecycle. Examples include `Pending`, `Running`, `Succeeded`, `Failed` and `Unknown`

# Pod Lifecycle
Pods follow a defined lifecycle, starting in the `Pending` [phase](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#pod-phase), moving through `Running` if at least one of its primary containers starts OK, and then through either the `Succeeded` or `Failed` phases depending on whether any container in the Pod terminated in failure.

Like individual application containers, Pods are considered to be relatively ephemeral (rather than durable) entities. Pods are created, assigned a unique ID ([UID](https://kubernetes.io/docs/concepts/overview/working-with-objects/names/#uids)), and scheduled to run on nodes where they remain until termination (according to restart policy) or deletion. If a [Node](https://kubernetes.io/docs/concepts/architecture/nodes/) dies, the Pods running on (or scheduled to run on) that node are [marked for deletion](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#pod-garbage-collection).
The control plane marks the Pods for removal after a timeout period.

## Pod Phase
A Pod's `status` field is a [PodStatus](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.33/#podstatus-v1-core) object, which has a `phase` field.

Here are the possible values for `phase`:

| Value       | Description                                                                                                                                                                                                                                                        |
| ----------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `Pending`   | The Pod has been accepted by the Kubernetes cluster, but one or more of the containers has not been set up and made ready to run. This includes time a Pod spends waiting to be scheduled as well as the time spent downloading container images over the network. |
| `Running`   | The Pod has been bound to a node, and all of the containers have been created. At least one container is still running, or is in the process of starting or restarting.                                                                                            |
| `Succeeded` | All containers in the Pod have terminated in success, and will not be restarted.                                                                                                                                                                                   |
| `Failed`    | All containers in the Pod have terminated, and at least one container has terminated in failure. That is, the container either exited with non-zero status or was terminated by the system, and is not set for automatic restarting.                               |
| `Unknown`   | For some reason the state of the Pod could not be obtained. This phase typically occurs due to an error in communicating with the node where the Pod should be running.                                                                                            |

### Note:
When a pod is failing to start repeatedly, `CrashLoopBackOff` may appear in the `Status` field of some kubectl commands. Similarly, when a pod is being deleted, `Terminating` may appear in the `Status` field of some kubectl commands.

Make sure not to confuse _Status_, a kubectl display field for user intuition, with the pod's `phase`. Pod phase is an explicit part of the Kubernetes data model and of the [Pod API](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/pod-v1/).

```
  NAMESPACE               NAME               READY   STATUS             RESTARTS   AGE
  alessandras-namespace   alessandras-pod    0/1     CrashLoopBackOff   200        2d9h
```

---

A Pod is granted a term to terminate gracefully, which defaults to 30 seconds. You can use the flag `--force` to [terminate a Pod by force](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#pod-termination-forced).

Since Kubernetes 1.27, the kubelet transitions deleted Pods, except for [static Pods](https://kubernetes.io/docs/tasks/configure-pod-container/static-pod/) and [force-deleted Pods](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#pod-termination-forced) without a finalizer, to a terminal phase (`Failed` or `Succeeded` depending on the exit statuses of the pod containers) before their deletion from the API server.

If a node dies or is disconnected from the rest of the cluster, Kubernetes applies a policy for setting the `phase` of all Pods on the lost node to Failed.


## Garbage collection of Pods
For failed Pods, the API objects remain in the cluster's API until a human or [controller](https://kubernetes.io/docs/concepts/architecture/controller/) process explicitly removes them.

The Pod garbage collector (PodGC), which is a controller in the control plane, cleans up terminated Pods (with a phase of `Succeeded` or `Failed`), when the number of Pods exceeds the configured threshold (determined by `terminated-pod-gc-threshold` in the kube-controller-manager). This avoids a resource leak as Pods are created and terminated over time.

Additionally, PodGC cleans up any Pods which satisfy any of the following conditions:

1. are orphan Pods - bound to a node which no longer exists,
2. are unscheduled terminating Pods,
3. are terminating Pods, bound to a non-ready node tainted with [`node.kubernetes.io/out-of-service`](https://kubernetes.io/docs/reference/labels-annotations-taints/#node-kubernetes-io-out-of-service).

Along with cleaning up the Pods, PodGC will also mark them as failed if they are in a non-terminal phase. Also, PodGC adds a Pod disruption condition when cleaning up an orphan Pod. See [Pod disruption conditions](https://kubernetes.io/docs/concepts/workloads/pods/disruptions/#pod-disruption-conditions) for more details.


# Pod lifetime
Whilst a Pod is running, the kubelet is able to restart containers to handle some kind of faults. Within a Pod, Kubernetes tracks different container [states](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#container-states) and determines what action to take to make the Pod healthy again.

In the Kubernetes API, Pods have both a specification and an actual status. The status for a Pod object consists of a set of [Pod conditions](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#pod-conditions). You can also inject [custom readiness information](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#pod-readiness-gate) into the condition data for a Pod, if that is useful to your application.

Pods are only [scheduled](https://kubernetes.io/docs/concepts/scheduling-eviction/) once in their lifetime; assigning a Pod to a specific node is called _binding_, and the process of selecting which node to use is called _scheduling_. Once a Pod has been scheduled and is bound to a node, Kubernetes tries to run that Pod on the node. The Pod runs on that node until it stops, or until the Pod is [terminated](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#pod-termination); if Kubernetes isn't able to start the Pod on the selected node (for example, if the node crashes before the Pod starts), then that particular Pod never starts.

You can use [Pod Scheduling Readiness](https://kubernetes.io/docs/concepts/scheduling-eviction/pod-scheduling-readiness/) to delay scheduling for a Pod until all its _scheduling gates_ are removed. For example, you might want to define a set of Pods but only trigger scheduling once all the Pods have been created.

## Pods and fault recovery
If one of the containers in the Pod fails, then Kubernetes may try to restart that specific container. Read [How Pods handle problems with containers](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#container-restarts) to learn more.

Pods can however fail in a way that the cluster cannot recover from, and in that case Kubernetes does not attempt to heal the Pod further; instead, Kubernetes deletes the Pod and relies on other components to provide automatic healing.

If a Pod is scheduled to a [node](https://kubernetes.io/docs/concepts/architecture/nodes/) and that node then fails, the Pod is treated as unhealthy and Kubernetes eventually deletes the Pod. A Pod won't survive an [eviction](https://kubernetes.io/docs/concepts/scheduling-eviction/) due to a lack of resources or Node maintenance.

Kubernetes uses a higher-level abstraction, called a [controller](https://kubernetes.io/docs/concepts/architecture/controller/), that handles the work of managing the relatively disposable Pod instances.

A given Pod (as defined by a UID) is never "rescheduled" to a different node; instead, that Pod can be replaced by a new, near-identical Pod. If you make a replacement Pod, it can even have same name (as in `.metadata.name`) that the old Pod had, but the replacement would have a different `.metadata.uid` from the old Pod.

Kubernetes does not guarantee that a replacement for an existing Pod would be scheduled to the same node as the old Pod that was being replaced.

## Associated lifetimes
When something is said to have the same lifetime as a Pod, such as a [volume](https://kubernetes.io/docs/concepts/storage/volumes/), that means that the thing exists as long as that specific Pod (with that exact UID) exists. If that Pod is deleted for any reason, and even if an identical replacement is created, the related thing (a volume, in this example) is also destroyed and created anew.


# Container states
As well as the [phase](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#pod-phase) of the Pod overall, Kubernetes tracks the state of each container inside a Pod. You can use [container lifecycle hooks](https://kubernetes.io/docs/concepts/containers/container-lifecycle-hooks/) to trigger events to run at certain points in a container's lifecycle.

Once the [scheduler](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-scheduler/) assigns a Pod to a Node, the kubelet starts creating containers for that Pod using a [container runtime](https://kubernetes.io/docs/setup/production-environment/container-runtimes). There are three possible container states: `Waiting`, `Running`, and `Terminated`.

To check the state of a Pod's containers, you can use `kubectl describe pod <name-of-pod>`. The output shows the state for each container within that Pod.

Each state has a specific meaning:

## Waiting
If a container is not in either the `Running` or `Terminated` state, it is `Waiting`. A container in the `Waiting` state is still running the operations it requires in order to complete start up: for example, pulling the container image from a container image registry, or applying [Secret](https://kubernetes.io/docs/concepts/configuration/secret/) data. When you use `kubectl` to query a Pod with a container that is `Waiting`, you also see a Reason field to summarize why the container is in that state.

## Running
The `Running` status indicates that a container is executing without issues. If there was a `postStart` hook configured, it has already executed and finished. When you use `kubectl` to query a Pod with a container that is `Running`, you also see information about when the container entered the `Running` state.

## Terminated
A container in the `Terminated` state began execution and then either ran to completion or failed for some reason. When you use `kubectl` to query a Pod with a container that is `Terminated`, you see a reason, an exit code, and the start and finish time for that container's period of execution.

If a container has a `preStop` hook configured, this hook runs before the container enters the `Terminated` state.

