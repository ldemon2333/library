A **Kubeconfig is a YAML file** with all the [Kubernetes cluster](https://devopscube.com/setup-kubernetes-cluster-kubeadm/) details, certificates, and secret tokens to authenticate the cluster.

When you use `kubectl`, it uses the information in the kubeconfig file to connect to the kubernetes cluster API. The default location of the Kubeconfig file is `$HOME/.kube/config`
![[Pasted image 20250616124608.png]]
Not just users, any cluster component that need to authenticate to the kubernetes API server need the kubeconfig file.

For example, [kubernetes cluster components](https://devopscube.com/kubernetes-architecture-explained/) like controller manager, scheduler and kubelet use the kubeconfig files to interact with the API server.

All the cluster Kubeconfig files are present in the control plane `**/etc/kubernetes**` folder (`.conf` files).

# Example Kubeconfig File
Here is an example of a Kubeconfig. It needs the following key information to connect to the Kubernetes clusters.

1. **certificate-authority-data**: Cluster CA
2. **server**: Cluster endpoint (IP/DNS of the master node)
3. **name**: Cluster name
4. **user**: name of the user/service account.
5. **token**: Secret token of the user/service account.
```yaml
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: <ca-data-here>
    server: https://your-k8s-cluster.com
  name: <cluster-name>
contexts:
- context:
    cluster:  <cluster-name>
    user:  <cluster-name-user>
  name:  <cluster-name>
current-context:  <cluster-name>
kind: Config
preferences: {}
users:
- name:  <cluster-name-user>
  user:
    token: <secret-token-here>
```



