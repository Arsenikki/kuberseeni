# Edge Computing Cluster with Industrial IoT Capabilities
This project is created as a part of my master's thesis and it was used to evaluate a proof of concept industrial IoT edge computing system, which is based on k3s Kubernetes distribution. The main features are listed below: 

1. Automated Kubernetes cluster initialization using Ansible, which handles:
    * Prerequisite changes in OS before k3s installation
    * Installation of lightweight k3s Kubernetes distribution 
    * MetalLB load balancer configuration
2. GitOps-based approach for the application lifecycle management, including: 
    * Flux CD syncs the application configurations from GitHub
    * Helm Operator automatically installs/updates the manifests in the cluster
    * Flux CD also watches container registry for changes and automatically updates the configuration files in Git with the new image
3. VerneMQ distributed MQTT broker workload, including:
    * Using the previous GitOps method for simple installation
    * Load balancer + Traefik setup to distribute load across all the nodes
4. Performance testing using mqtt-benchmark tool
    * Mimics IoT use cases with 100 client devices, each sending 1000 messages with QoS 0 & QoS 1
    * Prometheus + Grafana used for system resource usage monitoring

## System Installation
The ubuntu configuration section is a pre-requisite for the K3s Ansible Playbook, which automatically configures the cluster nodes and installs k3s Kubernetes distribution with some minor changes to match our use-case.

### Ubuntu Configuration


Firstly, a static IP address is defined to make it easier to automate the installation steps and to connect external devices to the MQTT broker application with a fixed IP. By default, a DHCP is enabled in netplan configuration, resulting in the DHCP server assigning the IP address dynamically for each node. The static IP address can be configured by changing a netplan configuration. An example is shown below, which defines our master node.

```
network:
version: 2
ethernets:
    eth0:
        dhcp4: false
        addresses: [192.168.1.100/24]
        gateway4: 192.168.1.1
        nameservers:
            addresses: [8.8.4.4,8.8.8.8]
```

Secondly, Cgroups are required to be enabled to execute containerized workloads in Linux OS. Using Ubuntu 20.04, this is done by appending the cgroup flags in /boot/firmware/cmdline.txt as shown:

```
cgroup_enable=cpuset cgroup_memory=1 cgroup_enable=memory
```

Lastly, passwordless SSH access needs to be configured as it's a requirement for the selected Ansible installation approach. This is done by generating a key pair on the developer's pc. The private key is kept securely on the developer's pc and the same public key needs to be copied on all cluster nodes. This can be done with OpenSSH, using the ssh-copy-id tool by specifying the file path of the public key, and the user and IP address of the particular remote machine. The example below shows how one of the nodes is configured. 

```
ssh-copy-id -i ~/.ssh/id_rsa.pub maintainer@192.168.1.100
```

### K3s Ansible Playbook with MetalLB Configuration


The modified k3s Ansible Playbook is available at: https://github.com/Arsenikki/kuberseeni/tree/master/k3s-ansible. There are some necessary fixes, which make it work with the ARM64-based system running on Ubuntu 20.04. K3s is also defined to skip serviceLB load balancer installation with an additional install argument. Instead, the MetalLB load balancer is used, and therefore an extra step needs to be added to automatically configure the MetalLB after k3s installation is completed. It downloads the helm chart from a public repo and uses the built-in CRD controller to install it. The chart is available at: https://github.com/Arsenikki/kuberseeni/blob/master/metallb/helmChart.yaml.


## Configuring Applications
This sections describes the installation of Flux CD GitOps tool and how it's used to install VerneMQ workload. 
### Flux CD and Flux Helm Operator

Flux CD is installed using Helm, and a remote repository, path inside the repository, ARM64 Docker image, and sync intervals are specified with additional arguments: 

```
helm upgrade -i flux fluxcd/flux --namespace flux \
--set git.url=git@github.com:Arsenikki/kuberseeni \
--set git.path=helmReleases \
--set image.repository=docker.io/raspbernetes/flux \
--set image.tag=latest \
--set git.pollInterval=1m
```

Next, HelmRelease CRDs are configured using kubectl, which are used by the Helm Operator to install the wanted Helm charts.

```
kubectl apply -f https://raw.githubusercontent.com/fluxcd/helm-operator/1.1.0/deploy/crds.yaml
```

Helm Operator is installed using Helm, and the helm version, ARM64-based Docker image, git secret, and sync intervals are specified with additional arguments:

```
helm upgrade -i helm-operator fluxcd/helm-operator --namespace flux \
--set helm.versions=v3 \
--set image.repository=docker.io/raspbernetes/helm-operator \
--set image.tag=latest \
--set git.ssh.secretName=flux-git-deploy \
--set chartsSyncInterval=1m
```

Lastly, Flux generates an SSH key at startup and the public key can be fetched using fluxctl CLI tool as shown below, which then needs to be then added to the Github repository as a deployment key.

```
fluxctl identity --k8s-fwd-ns flux
```

Now the Flux CD GitOps tool is configured to synchronize the files from the specified repository path to the cluster. Helm-operator will automatically install them after that. The next section shows how an application is configured using this approach.

### VerneMQ configuration

HelmRelease CRD is used to configure the application workload, which is very similar method to how Helm uses values files for configuration. The CRD for VerneMQ is available at: https://github.com/Arsenikki/kuberseeni/blob/master/helmReleases/vernemqCRD.yaml, which has a reference to the wanted Helm chart and configured values, which override the default values included in the Helm chart. Here, the referenced chart had to be slightly modified from original to support automatic discovery, and therefore the modified chart is available here: https://github.com/Arsenikki/kuberseeni/tree/master/vernemq.
