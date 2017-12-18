# Provisioning Compute Resources

Kubernetes requires a set of machines to host the Kubernetes control plane and the worker nodes where containers are ultimately run. In this lab you will provision the compute resources required for running a secure and highly available Kubernetes cluster across a single [compute zone](https://cloud.google.com/compute/docs/regions-zones/regions-zones).

> Ensure a default compute zone and region have been set as described in the [Prerequisites](01-prerequisites.md#set-a-default-compute-region-and-zone) lab.

## Networking

The Kubernetes [networking model](https://kubernetes.io/docs/concepts/cluster-administration/networking/#kubernetes-model) assumes a flat network in which containers and nodes can communicate with each other. In cases where this is not desired [network policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/) can limit how groups of containers are allowed to communicate with each other and external network endpoints.

> Setting up network policies is out of scope for this tutorial.

### Virtual Private Cloud Network

In this section a dedicated [Virtual Private Cloud](https://cloud.google.com/compute/docs/networks-and-firewalls#networks) (VPC) network will be setup to host the Kubernetes cluster.

#### GCP

Create the `kubernetes-the-hard-way` custom VPC network:

```
gcloud compute networks create kubernetes-the-hard-way --mode custom
```

#### Openstack

Create a private network:

```
openstack network create kubernetes-the-hard-way
```

Note: on my host (OVH) this returns `'NoneType' object is not iterable` but it creates the network anyway.


A [subnet](https://cloud.google.com/compute/docs/vpc/#vpc_networks_and_subnets) must be provisioned with an IP address range large enough to assign a private IP address to each node in the Kubernetes cluster.

#### GCP

Create the `kubernetes` subnet in the `kubernetes-the-hard-way` VPC network:

```
gcloud compute networks subnets create kubernetes \
  --network kubernetes-the-hard-way \
  --range 10.240.0.0/24
```

#### Openstack

Create the `kubernetes` subnet on the `kubernetes-the-hard-way` private network.

```
openstack subnet create kubernetes --network kubernetes-the-hard-way --subnet-range 10.240.0.0/24 --no-gateway
```

Note: Again this returns `'NoneType' object is not iterable`, but actually works perfectly


> The `10.240.0.0/24` IP address range can host up to 254 compute instances.

### Firewall Rules

Create a firewall rule that allows internal communication across all protocols:

#### GCP

```
gcloud compute firewall-rules create kubernetes-the-hard-way-allow-internal \
  --allow tcp,udp,icmp \
  --network kubernetes-the-hard-way \
  --source-ranges 10.240.0.0/24,10.200.0.0/16
```

#### Openstack

```
openstack security group create kubernetes-the-hard-way-allow-internal
openstack security group rule create --proto tcp --remote-group=kubernetes-the-hard-way-allow-internal --dst-port 1:65525 kubernetes-the-hard-way-allow-internal
openstack security group rule create --proto udp --remote-group=kubernetes-the-hard-way-allow-internal --dst-port 1:65525 kubernetes-the-hard-way-allow-internal
openstack security group rule create --proto icmp --remote-group=kubernetes-the-hard-way-allow-internal kubernetes-the-hard-way-allow-internal
```



Create a firewall rule that allows external SSH, ICMP, and HTTPS:

#### GCP

```
gcloud compute firewall-rules create kubernetes-the-hard-way-allow-external \
  --allow tcp:22,tcp:6443,icmp \
  --network kubernetes-the-hard-way \
  --source-ranges 0.0.0.0/0
```

#### Openstack

```
openstack security group create kubernetes-the-hard-way-allow-external
openstack security group rule create --proto tcp --dst-port=22 kubernetes-the-hard-way-allow-external
openstack security group rule create --proto tcp --dst-port=6443 kubernetes-the-hard-way-allow-external
```

> An [external load balancer](https://cloud.google.com/compute/docs/load-balancing/network/) will be used to expose the Kubernetes API Servers to remote clients.

List the firewall rules in the `kubernetes-the-hard-way` VPC network:

```
gcloud compute firewall-rules list --filter "network: kubernetes-the-hard-way"
```

> output

```
NAME                                         NETWORK                  DIRECTION  PRIORITY  ALLOW                 DENY
kubernetes-the-hard-way-allow-external       kubernetes-the-hard-way  INGRESS    1000      tcp:22,tcp:6443,icmp
kubernetes-the-hard-way-allow-internal       kubernetes-the-hard-way  INGRESS    1000      tcp,udp,icmp
```

### Kubernetes Public IP Address

Allocate a static IP address that will be attached to the external load balancer fronting the Kubernetes API Servers:

```
gcloud compute addresses create kubernetes-the-hard-way \
  --region $(gcloud config get-value compute/region)
```

Verify the `kubernetes-the-hard-way` static IP address was created in your default compute region:

```
gcloud compute addresses list --filter="name=('kubernetes-the-hard-way')"
```

> output

```
NAME                     REGION    ADDRESS        STATUS
kubernetes-the-hard-way  us-west1  XX.XXX.XXX.XX  RESERVED
```

## Compute Instances

The compute instances in this lab will be provisioned using [Ubuntu Server](https://www.ubuntu.com/server) 16.04, which has good support for the [cri-containerd container runtime](https://github.com/kubernetes-incubator/cri-containerd). Each compute instance will be provisioned with a fixed private IP address to simplify the Kubernetes bootstrapping process.

### Kubernetes Controllers

Create three compute instances which will host the Kubernetes control plane:

#### GCP

```
for i in 0 1 2; do
  gcloud compute instances create controller-${i} \
    --async \
    --boot-disk-size 200GB \
    --can-ip-forward \
    --image-family ubuntu-1604-lts \
    --image-project ubuntu-os-cloud \
    --machine-type n1-standard-1 \
    --private-network-ip 10.240.0.1${i} \
    --scopes compute-rw,storage-ro,service-management,service-control,logging-write,monitoring \
    --subnet kubernetes \
    --tags kubernetes-the-hard-way,controller
done
```

#### Openstack

You will need to collect some information specific to your cloud first.

```
IMAGEID=$(glance image-list | grep 'Ubuntu 16.04' | awk '{print $2}')
```

Note: You may need to modify that based on the output of `glance image-list` to ensure you get the correct image.

FLAVORID=$(nova flavor-list | grep 'vps-ssd-1' | awk '{print $2}')

```
for i in 0 1 2
do
  nova boot --image='Ubuntu 16.04' --flavor=vps-ssd-1 \
    --meta kubetype=controller \
    --nic net-name=Ext-Net \
    --nic net-name=kubernetes-the-hard-way,v4-fixed-ip=10.240.0.1${i} \
    --security-groups=kubernetes-the-hard-way-allow-internal,kubernetes-the-hard-way-allow-external \
    --user-data scripts/controller-userdata.yml --key-name chris controller-${i}
done
```

### Kubernetes Workers

Each worker instance requires a pod subnet allocation from the Kubernetes cluster CIDR range. The pod subnet allocation will be used to configure container networking in a later exercise. The `pod-cidr` instance metadata will be used to expose pod subnet allocations to compute instances at runtime.

> The Kubernetes cluster CIDR range is defined by the Controller Manager's `--cluster-cidr` flag. In this tutorial the cluster CIDR range will be set to `10.200.0.0/16`, which supports 254 subnets.

Create three compute instances which will host the Kubernetes worker nodes:

#### GCP

```
for i in 0 1 2; do
  gcloud compute instances create worker-${i} \
    --async \
    --boot-disk-size 200GB \
    --can-ip-forward \
    --image-family ubuntu-1604-lts \
    --image-project ubuntu-os-cloud \
    --machine-type n1-standard-1 \
    --metadata pod-cidr=10.200.${i}.0/24 \
    --private-network-ip 10.240.0.2${i} \
    --scopes compute-rw,storage-ro,service-management,service-control,logging-write,monitoring \
    --subnet kubernetes \
    --tags kubernetes-the-hard-way,worker
done
```

#### Openstack

```
for i in 0 1 2
do
  nova boot --image='Ubuntu 16.04' --flavor=vps-ssd-1 \
    --nic net-name=Ext-Net \
    --nic net-name=kubernetes-the-hard-way,v4-fixed-ip=10.240.0.2${i} \
    --security-groups=kubernetes-the-hard-way-allow-internal,kubernetes-the-hard-way-allow-external \
    --meta pod-cidr=10.200.${i}.0/24 --meta kubetype=worker \
    --user-data scripts/controller-userdata.yml --key-name chris worker-${i}
done
```

### Verification

List the compute instances in your default compute zone:

#### GCP

```
gcloud compute instances list
```

> output

```
NAME          ZONE        MACHINE_TYPE   PREEMPTIBLE  INTERNAL_IP  EXTERNAL_IP     STATUS
controller-0  us-west1-c  n1-standard-1               10.240.0.10  XX.XXX.XXX.XXX  RUNNING
controller-1  us-west1-c  n1-standard-1               10.240.0.11  XX.XXX.X.XX     RUNNING
controller-2  us-west1-c  n1-standard-1               10.240.0.12  XX.XXX.XXX.XX   RUNNING
worker-0      us-west1-c  n1-standard-1               10.240.0.20  XXX.XXX.XXX.XX  RUNNING
worker-1      us-west1-c  n1-standard-1               10.240.0.21  XX.XXX.XX.XXX   RUNNING
worker-2      us-west1-c  n1-standard-1               10.240.0.22  XXX.XXX.XX.XX   RUNNING
```

#### Openstack

```
nova list --name="worker*|controller*" --minimal
```

> output

```
+--------------------------------------+--------------+
| ID                                   | Name         |
+--------------------------------------+--------------+
| 42f45d04-72af-499f-967a-3fc6d48e6661 | controller-0 |
| e527bfdc-dd30-41f1-9925-b32619131bb5 | controller-1 |
| f22b602e-ab6d-4bd8-bb77-0bd01ebc904f | controller-2 |
| a43775e2-88e1-4c0e-83da-53068e2cf153 | worker-0     |
| aa138594-9fd3-4743-9d9b-d63d6bea0cf6 | worker-1     |
| 2d406138-9166-494b-bcb5-a7b0c46b4756 | worker-2     |
+--------------------------------------+--------------+

```

Next: [Provisioning a CA and Generating TLS Certificates](04-certificate-authority.md)
