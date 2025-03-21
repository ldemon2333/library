This section is relevant for people adopting a new built-in [sidecar containers](https://kubernetes.io/docs/concepts/workloads/pods/sidecar-containers/) feature for their workloads.

Sidecar container is not a new concept as posted in the [blog post](https://kubernetes.io/blog/2015/06/the-distributed-system-toolkit-patterns/). Kubernetes allows running multiple containers in a Pod to implement this concept. However, running a sidecar container as a regular container has a lot of limitations being fixed with the new built-in sidecar containers support.

# Objectives
- Understand the need for sidecar containers
- Be able to troubleshoot issues with the sidecar containers
- Understand options to universally "inject" sidecar containers to any workload

# Sidecar containers overview
Sidecar containers are secondary containers that run along with the main application container within the same Pod. These containers are used to enhance or to extend the functionality of the primary app container by providing additional services, or functionalities such as logging, monitoring, security, or data synchronization, without directly altering the primary application code.

The concept of sidecar containers is not new and there are multiple implementations of this concept. As well as sidecar containers that you, the person defining the Pod, want to run, you can also find that some [addons](https://kubernetes.io/docs/concepts/cluster-administration/addons/) modify Pods - before the Pods start running - so that there are extra sidecar containers. The mechanisms to _inject_ those extra sidecars are often [mutating webhooks](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#mutatingadmissionwebhook). For example, a service mesh addon might inject a sidecar that configures mutual TLS and encryption in transit between different Pods.

While the concept of sidecar containers is not new, the native implementation of this feature in Kubernetes, however, is new. And as with every new feature, adopting this feature may present certain challenges.

This tutorial explores challenges and solutions that can be experienced by end users as well as by authors of sidecar containers.

# Benefits of a built-in sidecar container
Using Kubernetes' native support for sidecar containers provides several benefits:

1. You can configure a native sidecar container to start ahead of [init containers](https://kubernetes.io/docs/concepts/workloads/pods/init-containers/).
2. The built-in sidecar containers can be authored to guarantee that they are terminated last. Sidecar containers are terminated with a `SIGTERM` signal once all the regular containers are completed and terminated. If the sidecar container isn’t gracefully shut down, a `SIGKILL` signal will be used to terminate it.
3. With Jobs, when Pod's `restartPolicy: OnFailure` or `restartPolicy: Never`, native sidecar containers do not block Pod completion. With legacy sidecar containers, special care is needed to handle this situation.
4. Also, with Jobs, built-in sidecar containers would keep being restarted once they are done, even if regular containers would not with Pod's `restartPolicy: Never`.

See [differences from init containers](https://kubernetes.io/docs/concepts/workloads/pods/sidecar-containers/#differences-from-application-containers) to learn more about it.

# Adopting built-in sidecar containers
The `SidecarContainers` [feature gate](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/) is in beta state starting from Kubernetes version 1.29 and is enabled by default. Some clusters may have this feature disabled or have software installed that is incompatible with the feature.

When this happens, the Pod may be rejected or the sidecar containers may block Pod startup, rendering the Pod useless. This condition is easy to detect as the Pod simply gets stuck on initialization. However, it is often unclear what caused the problem.

Here are the considerations and troubleshooting steps that one can take while adopting sidecar containers for their workload.

## Ensure the feature gate is enabled


https://kubernetes.io/docs/tutorials/configuration/pod-sidecar-containers/