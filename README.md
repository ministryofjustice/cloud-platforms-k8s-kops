# cloud-platforms-k8s-kops
This project is a Kubernetes testbed, intended to allow experimentation and evaluation with minimal setup required - essentially a prototype hosting platform, or Rails scaffold for Kubernetes.

Included here are resources to provision Kubernetes clusters with [Kops](https://github.com/kubernetes/kops), as a standin for AWS Elastic Kubernetes Service until that becomes available.

Also included are sample cluster components for common services (authentication, HTTP routing, DNS, etc), and test applications for deployment; the expectation is that all cluster components will be able to be deployed to an EKS cluster as-is.

### Sections

This readme is split into the following sections:

- [Cluster Operations](#cluster-operations)
- [Authentication](#authentication)
- [Cluster Components](#cluster-components)


### GPG Note
As this repo uses `git-crypt` your GPG key must be added to this repo for you to be able to access sensitive information such as SSH keys and authentication secrets.

A GPG key is not required to obtain Kubernetes credentials and interact with clusters however - cluster admin credentials can be obtained by logging into [Kuberos](#authentication) with your Github account. Additionally, as long as you have valid AWS credentials and access to the Kops S3 bucket you can download static admin credentials for `kubeconfig` using `kops`.

## Cluster operations
Info on how to modify infrastructure and configuration of existing clusters, and how to provision new clusters.

### Prerequisites
- Install [Kops](https://github.com/kubernetes/kops)
- Install [kubectl](https://kubernetes.io/docs/reference/kubectl/overview/)
- Install [git-crypt](https://www.agwa.name/projects/git-crypt/)

#### MacOS
```
brew install kops
brew install kubernetes-cli
brew install git-crypt
```

### Working with existing cluster (`cluster1`)

#### Prerequisites
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

### Editing cluster config / changing config

Edit the `cluster.yml` with your desired changes (e.g. [adding an additional IAM policy for external-dns](#iam-policies-for-external-dns)), then run

```
$ kops replace -f $YOUR_CLUSTER.yml
```

this will replace the existing cluster specification YML in S3 with your new version. Then:

```
$ kops update cluster
```

to preview the changes kops will apply to AWS. Then:

```
$ kops update cluster --yes
```

to actually apply the changes. If these changes require that EC2 instances be replaced this will be noted in the output; a [rolling update](https://github.com/kubernetes/kops/blob/master/docs/cli/kops_rolling-update.md) can then be performed with:

```
$ kops rolling-update cluster
```

to preview changes, and:

```
$ kops rolling-update cluster --yes
```

to perform the rolling update.

### Creating a new cluster
Set an environment variable for your cluster name (must be DNS compliant - no underscores or similar):

`export CLUSTER_NAME=cluster2`

#### Create a Route53 HostedZone

Both `kops` and the [external-dns component](#external-dns) will create DNS records for the cluster, so a cluster-specific DNS zone should be created to avoid interfering with other clusters or services in AWS.

1. Using either the console or AWS CLI create a Route53 hosted zone called `$CLUSTER_NAME.kops.integration.dsd.io.`
2. Copy the value of the `NS` record for that zone (the 4 lines like `ns-1722.awsdns-23.co.uk.`)
3. In the parent HostedZone, `kops.integration.dsd.io.`, create an `NS` record containing those 4 nameservers

#### Create cluster specification
Generate a new SSH key and use `kops` to generate a new cluster specification - example for `cluster1.yaml`:

```
export KOPS_STATE_STORE=s3://moj-cloud-platforms-kops-state-store

ssh-keygen -t rsa -f ssh/${CLUSTER_NAME}_kops_id_rsa -N '' -C ${CLUSTER_NAME}

kops create cluster \
    --name ${CLUSTER_NAME}.kops.integration.dsd.io \
    --zones=eu-west-1a,eu-west-1b,eu-west-1c \
    --node-size=t2.medium \
    --node-count=3 \
    --master-zones=eu-west-1a,eu-west-1b,eu-west-1c \
    --master-size=t2.medium \
    --topology=private \
    --dns-zone=${CLUSTER_NAME}.kops.integration.dsd.io \
    --ssh-public-key=ssh/${CLUSTER_NAME}_kops_id_rsa.pub \
    --authorization=RBAC \
    --networking=calico \
    --output=yaml \
    --bastion \
    --dry-run \
    > ${CLUSTER_NAME}.yaml
```

##### IAM policies for external-dns

For [external-dns](#external-dns) to be able to manage records in Route53, the EC2 instances in the cluster must have an IAM policy allowing Route53 changes. Because this functionality is not part of core Kubernetes, an additional policy must be added to the instances. Kops allows you to add additional policies in the cluster specification and have an IAM policy created and attached to the role. To do so, add this block to the `spec` section of your `cluster.yml` file (see `cluster1.yml` for a working example):

```
  additionalPolicies:
    node: |
      [
        {
          "Effect": "Allow",
          "Action": [
            "route53:ChangeResourceRecordSets"
          ],
          "Resource": [
            "arn:aws:route53:::hostedzone/$YOUR_ZONE_ID"
          ]
        },
        {
          "Effect": "Allow",
          "Action": [
            "route53:ListHostedZones",
            "route53:ListResourceRecordSets"
          ],
          "Resource": [
            "*"
          ]
        }
      ]
```

Where `$YOUR_ZONE_ID` is the ID of your Route53 zone for your cluster.

##### High availability, network topology and NAT gateways
The above command will create a highly-available cluster across all three AZs in eu-west-1, using a private network topology - 3 public subnets containing 3 NAT gateways and a single SSH bastion host, and 3 private subnets containing all masters and worker nodes. To run a smaller, non-HA cluster for testing, specify one AZ only for the `--zones` and `--master-zones` flag (ie. `--zones=eu-west-1a`. To reduce the number of EC2 instances provisioned, you can also specify `--topology=public` to deploy all instances to public subnets without NAT gateways, and remove the `--bastion` flag to skip provisioning of the SSH bastion host.

#### Create cluster specification in kops state store
`$ kops create -f ${CLUSTER_NAME}.yaml`

#### Create SSH public key in kops state store
`$ kops create secret --name ${CLUSTER_NAME}.kops.integration.dsd.io sshpublickey admin -i ssh/${CLUSTER_NAME}_kops_id_rsa.pub`

#### Create cluster resources in AWS
aka update cluster in AWS according to the yaml specification:
`$ kops update cluster ${CLUSTER_NAME}.kops.integration.dsd.io --yes`

It takes a few minutes for the cluster to deploy and come up - you can check progress with:

`$ kops validate cluster`

Once it reports `Your cluster ${CLUSTER_NAME}.kops.integration.dsd.io is ready` you can proceed to use `kubectl` to interact with the cluster.

#### Changing cluster configuration after creation
Refer to [Editing cluster config / changing config](#editing-cluster-config-changing-config)

## Authentication

### Identity provider and Kubernetes cluster config
For our use case, we want authentication and identity to be handled by Github, and to derive all cluster access control from Github teams - projects will be deployed into namespaces (e.g. `pvb-production`, `cla-staging`), and access to resources in those namespaces should be available to the appropriate teams only (e.g. `PVB` and `CLA` teams).

Kubernetes supports authentication from external identity providers, including group definition, via [OIDC](https://kubernetes.io/docs/admin/authentication/#openid-connect-tokens). Github however only support OAuth2 as an authentication method, so an identity broker is required to translate from OAuth2 to OIDC.

As work on MOJ's identity service is ongoing, a development [Auth0](https://www.auth0.com) account has been created to act as a standin in the meantime. That Auth0 account has been configured with a rule to add github team membership to the OIDC `id_token`, which Kubernetes can than map to a [Role or ClusterRole](https://kubernetes.io/docs/admin/authorization/rbac/#role-and-clusterrole) resource - this rule can be viewed in `authentication/auth0-rules/whitelist-github-org-and-add-teams-to-group-claim.js`.

For Kubernetes to consume the user and group information in the Auth0 `id_token`, configuration options for Auth0 and user/group mapping must be passed to the Kubernetes master servers. This config can be provided in the Kops cluster specification, e.g:

```
  kubeAPIServer:
    oidcClientID: nEmRfCMu80zLneVipmzevohG0ECL1Sig
    oidcIssuerURL: https://moj-cloud-platforms-dev.eu.auth0.com/
    oidcUsernameClaim: nickname
    oidcGroupsClaim: https://api.cluster1.kops.integration.dsd.io/groups
```

This is included in the full `cluster1.yaml` specification.

### End user authentication and credential setup
End users require credentials on their local machines in order to be able to use `kubectl`, `helm` etc. Those credentials contain information obtained from the identity provider - in our case Auth0 - which requires a browser-based authentication flow.

To handle user authentication and generation of cluster credentials, a webapp called [Kuberos](#kuberos) has been deployed at [https://kuberos.apps.cluster1.kops.integration.dsd.io](https://kuberos.apps.cluster1.kops.integration.dsd.io).

**Important note** - the instructions provided by Kuberos will either overwrite your local `kubectl` config entirely, or any existing config for this specific cluster, so if you have already obtained static cluster credentials with `kops export kubecfg`, you can instead reference the downloaded `kubecfg.yaml` credentials as flags to `kubectl`:

```
$ KUBECONFIG=/path/to/kubecfg.yaml \
	kubectl --context cluster1.kops.integration.dsd.io \
		get pods
```

## Cluster components

Example cluster components/services in `cluster-components/` - intended as a starting point for experimentation and discussion, but also providing some useful functionality.

For convenience, most cluster components are installed using `Helm`/`Tiller`, so that must be installed first. As Helm packages often require arguments to be provided at installation, a `values.yml` file has been provided for each component that is using Helm.

### New cluster config

The example cluster components and applications config contain references to cluster-specific domain names, so when creating a new cluster you should copy the contents of `cluster-components/cluster1` to `cluster-components/$YOUR_CLUSTER` and replace references to `cluster1.kops.integration.dsd.io` with `$YOUR_CLUSTER.kops.integration.dsd.io` in all YAML files before creating any of the components below. For helm-managed components (nginx-ingress, external-dns and kube-lego) these references are in their respective `values.yml` files; for `kuberos` and `example-apps/nginx`, these references are in `ingress.yml` in both cases.

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

`$ helm install nginx-ingress -f cluster-components/$YOUR_CLUSTER/nginx-ingress/values.yml`

This will deploy the nginx-ingress controller using the arguments specified in `values.yml`. By default, nginx-ingress specifies a `Service` with `type=LoadBalancer`, so in AWS it will automatically create, configure and manage an ELB.

### external-dns

[external-dns](https://github.com/kubernetes-incubator/external-dns) is a Kubernetes incubator project that automatically creates/updates/deletes DNS entries in Route53 based on declared hostnames in `Ingress` and `Service` objects. To install with Helm:

`$ helm install external-dns -f cluster-components/$YOUR_CLUSTER/external-dns/values.yml`

The configuration in `values.yml` allows DNS to be created for `Service` objects only, as these examples are using a single `nginx-ingress` service and ELB; that nginx-ingress `Service` object has a `external-dns.alpha.kubernetes.io/hostname` annotation to create a wildcard DNS record for that ELB.

### kube-lego
[kube-lego](https://github.com/jetstack/kube-lego) is an in-cluster service that watches the Kubernetes API for `Ingress` rules with SSL/TLS definitions, obtains TLS certificates from Let's Encrypt, and configures the ingress-controller with the obtained cert. To install with Helm:

`$ helm install kube-lego -f cluster-components/$YOUR_CLUSTER/kube-lego/values.yml`

For an Ingress rule to receive a TLS certificate and for the ingress controller to make use of it, the Ingress rule must contain a `kubernetes.io/tls-acme: "true"` annotation, and a `tls` block defining the `Secret` where the certificate is stored. See `example-apps/nginx/ingress.yml` for a working example.

### kuberos
[kuberos](https://github.com/negz/kuberos/) is a simple app to handle cluster credential generation when using OIDC [Authentication](#authentication) - it's not great, but good enough for now.

As no Helm chart is available for Kuberos YAML resource definitions have been created in `cluster-components/kuberos` instead. To create or update these resources, run:

`$ kubectl apply -f cluster-components/$YOUR_CLUSTER/kuberos/`
