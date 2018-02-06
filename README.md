# cloud-platforms-k8s-kops
Resources for Kubernetes clusters with Kops, and example cluster service resources (not Kops-specifc).

## Prerequisites
- Install [Kops](https://github.com/kubernetes/kops)
- Install [kubectl](https://kubernetes.io/docs/reference/kubectl/overview/)
- Install [git-crypt](https://www.agwa.name/projects/git-crypt/)

**GPG Note:** As this repo uses `git-crypt` your GPG key must be added to this repo for you to be able to use the SSH keys. A GPG is not required to obtain Kubernetes credentials and interact with clusters however - as long as you have valid AWS credentials and access to the Kops S3 bucket you can download credentials for `kubeconfig` using `kops`.

### MacOS
```
brew install kops
brew install kubernetes-cli
brew install git-crypt
```

## Working with existing cluster (`cluster1`)

### Prerequisites
1. Make sure you have credentials for the correct AWS account and that they are active:
 `$ export AWS_PROFILE=mojds-platforms-integration`
2. Kops stores resource specifications and credentials in an S3 bucket, which must be specified before using `kops`:
 `$ export KOPS_STATE_STORE=s3://moj-cloud-platforms-kops-state-store`
3. Download cluster credentials from S3 and configure `kubectl` for use:
 `kops export kubecfg cluster1.kops.integration.dsd.io`
4. Verify that you can interact with the cluster:
 `$ kubectl cluster-info`
5. Check what's running:
 `$ kubectl get pods --all-namespaces`

## Editing cluster config / changing config

Refer to [kops documentation](https://github.com/kubernetes/kops/blob/master/docs/changing_configuration.md)

## Creating a new cluster

### Create cluster specification
Generate a new SSH key and use `kops` to generate a new cluster specification - example for `cluster1.yaml`:

```
export KOPS_STATE_STORE=s3://moj-cloud-platforms-kops-state-store
export CLUSTER_NAME=cluster1

ssh-keygen -t rsa -f ssh/${CLUSTER_NAME}_kops_id_rsa -N '' -C ${CLUSTER_NAME}

kops create cluster \
    --name ${CLUSTER_NAME}.kops.integration.dsd.io \
    --zones=eu-west-1a,eu-west-1b,eu-west-1c \
    --node-size=t2.medium \
    --node-count=3 \
    --master-zones=eu-west-1a,eu-west-1b,eu-west-1c \
    --master-size=t2.medium \
    --topology=private \
    --dns-zone=kops.integration.dsd.io \
    --ssh-public-key=ssh/${CLUSTER_NAME}_kops_id_rsa.pub \
    --authorization=RBAC \
    --networking=calico \
    --output=yaml \
    --bastion \
    --dry-run \
    > ${CLUSTER_NAME}.yaml
```

### Create cluster specification in kops state store
`$ kops create -f ${CLUSTER_NAME}.yaml`

### Create SSH public key in kops state store
`$ kops create secret --name ${CLUSTER_NAME}.kops.integration.dsd.io sshpublickey admin -i ssh/${CLUSTER_NAME}_kops_id_rsa.pub`

### Create cluster resources in AWS
aka update cluster in AWS according to the yaml specification:
`$ kops update cluster ${CLUSTER_NAME}.kops.integration.dsd.io --yes`

It takes a few minutes for the cluster to deploy and come up - you can check progress with:

`$ kops validate cluster`

Once it reports `Your cluster ${CLUSTER_NAME}.kops.integration.dsd.io is ready` you can proceed to use `kubectl` to interact with the cluster.

## Cluster components

Example cluster components/services in `cluster-components/` - intended as a starting point for experimentation and discussion, but also providing some useful functionality.

Cluster components are installed using `Helm`/`Tiller`, so that must be installed first.

### Helm

[Helm](https://helm.sh) - the package manager for Kubernetes. Includes two components - `Helm`, the command line client, and `Tiller`, the server-side component.

- Install command line client: `$ brew install kubernetes-helm`
- Create a ServiceAccount and RoleBinding for Tiller: `$ kubectl apply -f cluster-components/helm/rbac-config.yml`
- Install Tiller, with ServiceAccount config: `$ helm init --service-account tiller`

This installs Tiller as a cluster-wide service, with `cluster-admin` permissions. These permissions are too broad for a production multi-tenant environment, and are used here for test purposes only - in real-world usage Tiller should be deployed into each tenant namespace, with permissions scoped to that namespaces only.

#### Using Helm
- `$ helm repo update` - update package list, a la `apt-get update`
- `$ helm search` - see what's available in the public repo
- `$ helm install wordpress` - install Wordpress

### nginx-ingress

An [ingress controller](https://kubernetes.io/docs/concepts/services-networking/ingress/#ingress-controllers) based on nginx. Ingress controllers serve as HTTP routers/proxies that watch the Kubernetes API for `Ingress` rules, and create routes/proxy configs to route traffic from the internet to services and pods.

`$ helm install nginx-ingress -f cluster-components/nginx-ingress/values.yml`

This will deploy the nginx-ingress controller using the arguments specified in `values.yml`. By default, nginx-ingress specifies a `Service` with `type=LoadBalancer`, so in AWS it will automatically create, configure and manage an ELB.

### external-dns

[external-dns](https://github.com/kubernetes-incubator/external-dns) is a Kubernetes incubator project that automatically creates/updates/deletes DNS entries in Route53 based on declared hostnames in `Ingress` and `Service` objects. To install with Helm:

`$ helm install external-dns -f cluster-components/external-dns/values.yml`

The configuration in `values.yml` allows DNS to be created for `Service` objects only, as these examples are using a single `nginx-ingress` service and ELB; that nginx-ingress `Service` object has a `external-dns.alpha.kubernetes.io/hostname` annotation to create a wildcard DNS record for that ELB.
