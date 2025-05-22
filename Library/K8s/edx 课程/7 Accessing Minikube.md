# kubectl Configuration File
To access the Kubernetes cluster, the **kubectl** client needs the control plane node endpoint and appropriate credentials to be able to securely interact with the API Server running on the control plane node. While starting Minikube, the startup process creates, by default, a configuration file, **config**, inside the **.kube** directory (often referred to as the **[kubeconfig](https://kubernetes.io/docs/concepts/configuration/organize-cluster-access-kubeconfig/)**), which resides in the user's **home** directory. The configuration file has all the connection details required by **kubectl**. By default, the **kubectl** binary parses this file to find the control plane node's connection endpoint, along with the required credentials. Multiple **kubeconfig** files can be configured with a single **kubectl** client. To look at the connection details, we can either display the content of the **~/.kube/config** file (on Linux) or run the following command (the output is redacted for readability):

Once **kubectl** is installed, we can display information about the Minikube Kubernetes cluster with the **kubectl cluster-info** command: 

**$ kubectl cluster-info**


