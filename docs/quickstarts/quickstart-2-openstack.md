# QuickStart 2 - OpenStack target environment

In this QuickStart unit, we'll be gathering information and performing preparatory steps to enable k0rdent (running on your management node) to manage clusters on OpenStack and deploying our first child cluster.

As noted in the [Guide to QuickStarts](./index.md#quickstart-prerequisites), you'll need access to project in an OpenStack cloud to complete this step. If you haven't yet created a management node and installed k0rdent, go back to [QuickStart 1 - Management node and cluster](quickstart-1-mgmt-node-and-cluster.md).

Note that if you have already done other QuickStart guides (Quickstart 2 - [Azure](quickstart-2-azure.md), [AWS](quickstart-2-aws.md)) you can  use the same management cluster, continuing here with steps to add the ability to manage clusters on OpenStack. The k0rdent management cluster can accommodate multiple provider and credential setups, enabling management of multiple infrastructures. And even if your management node is external to OpenStack (for example, it could be on an AWS EC2 Virtual Machine), as long as you permit outbound traffic to all IP addresses from the management node, this should work fine. A big benefit of k0rdent is that it provides a single point of control and visibility across multiple clusters on multiple clouds and infrastructures.

> NOTE:
> **CLOUD SECURITY 101**: k0rdent requires _some_ but not _all_ permissions
> to manage OpenStack &mdash; doing so via the [CAPO (ClusterAPI for OpenStack) provider](https://github.com/kubernetes-sigs/cluster-api-provider-openstack).

The best practice for using k0rdent with OpenStack (this pattern is repeated with other clouds and infrastructures) is to create a dedicated 'k0rdent user' or generate application credentials on your account with the particular permissions k0rdent and CAPO require. In this section, we'll create and configure application credentials in an existing OpenStack project.


## Install the OpenStack CLI

We'll use the OpenStack CLI to create application credentials for the k0rdent user, so we'll install it on our management node:

```shell
sudo apt install -y python3 python3-venv
python3 -m venv openstack
source openstack/bin/activate
pip install python-openstackclient
```

## Export your cloud credentials


=== "clouds.yaml"

    This file stores configuration to access openstack clouds and should be placed in the current directory or other [supported location](https://docs.openstack.org/python-openstackclient/latest/configuration/index.html#configuration-files). To obtain it you can visit the OpenStack Horizon Dashboard (API Access page > download OpenStack RC > OpenStack clouds.yaml File).

    ```yaml
    clouds:
      openstack:
        auth:
          auth_url: https://cloud.yourorg.net/
          project_name: example-project
          username: example-user
          password: example-pass
        region_name: RegionOne
        interface: public
        identity_api_version: 3
    ```

=== "openrc.sh"

    Openrc is a script that sets all the required environment variables for openstack client to access your cloud. To obtain it you can visit the OpenStack Horizon Dashboard (API Access page > download OpenStack RC > OpenStack RC File). This should provide you with `<project-name>-openrc.sh` file.

    source this file in your shell and provide a password used to authenticate with your cloud.
    ```shell
    source *-openrc.sh
    ```

    Enter your password in the prompt, it won't be printed while typing.
    ```console
    Please enter your OpenStack Password for project <project-name> as user <user>:
    ```

=== "Environment Variables"

    Alternatively you can specify all the required variables by hand, below you can see list of most commonly required variables filled with example values.

    ```shell
    export OS_AUTH_URL="https://keystone.example.net/"
    export OS_PROJECT_NAME="example-project"
    export OS_USERNAME="example-user"
    export OS_PASSWORD="example-pass"
    export OS_REGION_NAME="RegionOne"
    export OS_INTERFACE="public"
    export OS_IDENTITY_API_VERSION=3
    export OS_USER_DOMAIN_NAME="Default"
    export OS_PROJECT_DOMAIN_ID="default"
    ```

> NOTE:
> For all the alternative and/or more sophisticated authentication methods please consult [OpenStack CLI Documentation](https://docs.openstack.org/python-openstackclient/latest/cli/authentication.html).


## Create Application Credentials for k0rdent

Once you're authenticated to your OpenStack cloud 

```shell
openstack application credential create --secret '<secret>' k0rdentQuickstart
```
```console
+--------------+------------------------------------------------------------------+
| Field        | Value                                                            |
+--------------+------------------------------------------------------------------+
| description  | None                                                             |
| expires_at   | None                                                             |
| id           | 29e636d3eb5d20e3aa60601c101dca7c                                 |
| name         | k0rdentQuickstart                                                |
| project_id   | 665f815aa75ee00083f1824078eb6146                                 |
| roles        | member load-balancer_member                                      |
| secret       | <secret>                                                         |
| system       | None                                                             |
| unrestricted | False                                                            |
| user_id      | 3604f4bf7dd38c0b24b9a869568b432ad6eea4bedc9737c6e6612344b9ce397a |
+--------------+------------------------------------------------------------------+
```


## Create Application Credentials credentials secret on the management cluster

Next, we create a `Secret` containing created credentials for the k0rdent user and apply this to the management cluster running k0rdent, in the `kcm-system` namespace (important: if you use another namespace, k0rdent will be unable to read the credentials). To do this, create the following YAML in a file called `openstack-cloud-config.yaml`:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: openstack-cloud-config
  namespace: kcm-system
  labels:
    k0rdent.mirantis.com/component: "kcm"
stringData:
  clouds.yaml: |
    clouds:
      openstack:
        auth:
          auth_url: <OS_AUTH_URL>
          application_credential_id: <OS_APPLICATION_CREDENTIAL_ID> # project_id returned by openstack command
          application_credential_secret: <OS_APPLICATION_CREDENTIAL_SECRET> # secret supplied to openstack command
        region_name: <OS_REGION_NAME>
        interface: <OS_INTERFACE>
        identity_api_version: <OS_IDENTITY_API_VERSION>
        auth_type: v3applicationcredential
```

> NOTE: contents of this file follow the `clouds.yaml` format previously mentioned in [Install OpenStack CLI](#export-your-cloud-credentials).
> All of the other `OS_` variables can also be found in your shell environment variables after sourcing `openrc.sh` file.


Then apply this YAML to the management cluster as follows:

```shell
kubectl apply -f openstack-cloud-config.yaml
```

## Create the k0rdent Cluster Manager credential object

Now we create the k0rdent Cluster Manager credential object. As in prior steps, create a YAML file called `openstack-cluster-identity-cred.yaml`:

```yaml
apiVersion: k0rdent.mirantis.com/v1alpha1
kind: Credential
metadata:
  name: openstack-cluster-identity-cred
  namespace: kcm-system
spec:
  description: "OpenStack credentials"
  identityRef:
    apiVersion: v1
    kind: Secret
    name: openstack-cloud-config
    namespace: kcm-system
```

> NOTE:
> - `.spec.identityRef.kind` should be Secret.
> - `.spec.identityRef.name` must match the Secret you created in previous step.
> - `.spec.identityRef.namespace` must be the same as the Secret’s namespace (kcm-system).


Now apply this YAML to your management cluster:

```shell
kubectl apply -f openstack-cluster-identity-cred.yaml
```

## Create the k0rdent Cluster Identity resource template ConfigMap

Now we create the k0rdent Cluster Identity resource template `ConfigMap`. As in prior steps, create a YAML file called `openstack-cluster-identity-resource-template.yaml`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: openstack-cloud-config-resource-template
  namespace: kcm-system
  labels:
    k0rdent.mirantis.com/component: "kcm"
  annotations:
    projectsveltos.io/template: "true"
data:
  configmap.yaml: |
    {{- $cluster := .InfrastructureProvider -}}
    {{- $identity := (getResource "InfrastructureProviderIdentity") -}}

    {{- $clouds := fromYaml (index $identity "data" "clouds.yaml" | b64dec) -}}
    {{- if not $clouds }}
      {{ fail "failed to decode clouds.yaml" }}
    {{ end -}}

    {{- $openstack := index $clouds "clouds" "openstack" -}}

    {{- if not (hasKey $openstack "auth") }}
      {{ fail "auth key not found in openstack config" }}
    {{- end }}
    {{- $auth := index $openstack "auth" -}}

    {{- $auth_url := index $auth "auth_url" -}}
    {{- $app_cred_id := index $auth "application_credential_id" -}}
    {{- $app_cred_name := index $auth "application_credential_name" -}}
    {{- $app_cred_secret := index $auth "application_credential_secret" -}}

    {{- $network_id := $cluster.status.externalNetwork.id -}}
    {{- $network_name := $cluster.status.externalNetwork.name -}}
    ---
    apiVersion: v1
    kind: Secret
    metadata:
      name: openstack-cloud-config
      namespace: kube-system
    type: Opaque
    stringData:
      cloud.conf: |
        [Global]
        auth-url="{{ $auth_url }}"

        {{- if $app_cred_id }}
        application-credential-id="{{ $app_cred_id }}"
        {{- end }}

        {{- if $app_cred_name }}
        application-credential-name="{{ $app_cred_name }}"
        {{- end }}

        {{- if $app_cred_secret }}
        application-credential-secret="{{ $app_cred_secret }}"
        {{- end }}

        {{- if and (not $app_cred_id) (not $app_cred_secret) }}
        username="{{ index $openstack "username" }}"
        password="{{ index $openstack "password" }}"
        {{- end }}
        region="{{ index $openstack "region_name" }}"

        [LoadBalancer]
        {{- if $network_id }}
        floating-network-id="{{ $network_id }}"
        {{- end }}

        [Networking]
        {{- if $network_name }}
        public-network-name="{{ $network_name }}"
        {{- end }}
```

This config map is a template for generating and populating cloud.conf file in the deployed child cluster. It's used by operators to utlilize OpenStack resources by underlying cloud like cinder volumes as persistent volumes and octavia as load balancer.

Now apply this YAML to your management cluster:

```shell
kubectl apply -f openstack-cluster-identity-resource-template.yaml -n kcm-system
```

## List available cluster templates

k0rdent is now fully configured to manage OpenStack in specified project. To create a cluster, begin by listing the available `ClusterTemplate` objects provided with k0rdent:

```shell
kubectl get clustertemplate -n kcm-system
```

You'll see output resembling what's below. Grab the name of the AWS standalone cluster template in its present version (in the below example, that's `openstack-standalone-cp-{{{ extra.docsVersionInfo.k0rdentVersion }}}`):

```console
NAMESPACE    NAME                            VALID
kcm-system   adopted-cluster-{{{ extra.docsVersionInfo.k0rdentVersion }}}           true
kcm-system   aws-eks-{{{ extra.docsVersionInfo.k0rdentVersion }}}                   true
kcm-system   aws-hosted-cp-{{{ extra.docsVersionInfo.k0rdentVersion }}}             true
kcm-system   aws-standalone-cp-{{{ extra.docsVersionInfo.k0rdentVersion }}}         true
kcm-system   azure-aks-{{{ extra.docsVersionInfo.k0rdentVersion }}}                 true
kcm-system   azure-hosted-cp-{{{ extra.docsVersionInfo.k0rdentVersion }}}           true
kcm-system   azure-standalone-cp-{{{ extra.docsVersionInfo.k0rdentVersion }}}       true
kcm-system   openstack-standalone-cp-{{{ extra.docsVersionInfo.k0rdentVersion }}}   true
kcm-system   vsphere-hosted-cp-{{{ extra.docsVersionInfo.k0rdentVersion }}}         true
kcm-system   vsphere-standalone-cp-{{{ extra.docsVersionInfo.k0rdentVersion }}}     true
```

## Create your ClusterDeployment

Now, to deploy a cluster, create a YAML file called `my-openstack-clusterdeployment1.yaml`. We'll use this to create a `ClusterDeployment` object in k0rdent, representing the deployed cluster. The `ClusterDeployment` identifies for k0rdent the `ClusterTemplate` you want to use for cluster creation, the identity credential object you want to create it under (that of your k0rdent user), plus the region and instance types you want to use to host control plane and worker nodes:

```yaml
apiVersion: k0rdent.mirantis.com/v1alpha1
kind: ClusterDeployment
metadata:
  name: my-openstack-clusterdeployment1
  namespace: kcm-system
spec:
  template: openstack-standalone-cp-{{{ extra.docsVersionInfo.k0rdentVersion }}}
  credential: openstack-cluster-identity-cred
  config:
    controlPlaneNumber: 1
    workersNumber: 1
    controlPlane:
      flavor: m1.medium
      image:
        filter:
          name: ubuntu-22.04-x86_64
    worker:
      flavor: m1.medium
      image:
        filter:
          name: ubuntu-22.04-x86_64
    externalNetwork:
      filter:
        name: "public"
    authURL: <OS_AUTH_URL>
    identityRef:
      name: openstack-cloud-config
      cloudName: eu
      region: RegionOne
```

> NOTE:
> - Adjust flavor, image name, and authURL to match your OpenStack environment.
> - For more information about the config options, see the [OpenStack Template Parameters](/template-openstack).


## Apply the `ClusterDeployment` to deploy the cluster

Finally, we'll apply the `ClusterDeployment` YAML (`my-openstack-clusterdeployment1.yaml`) to instruct k0rdent to deploy the cluster:

```shell
kubectl apply -f my-openstack-clusterdeployment1.yaml
```

Kubernetes should confirm this:

```console
clusterdeployment.k0rdent.mirantis.com/my-openstack-clusterdeployment1 created
```

There will be a delay as the cluster finishes provisioning. You can watch the provisioning process with the following command:

```shell
kubectl -n kcm-system get clusterdeployment.k0rdent.mirantis.com openstack-openstack-clusterdeployment1 --watch
```

In a short while, you'll see output such as:

```console
NAME                              READY   STATUS
my-openstack-clusterdeployment1   True    ClusterDeployment is ready
```

## Obtain the cluster's kubeconfig

Now you can retrieve the cluster's `kubeconfig`:

```shell
kubectl -n kcm-system get secret my-openstack-clusterdeployment1-kubeconfig -o jsonpath='{.data.value}' | base64 -d > my-openstack-clusterdeployment1.kubeconfig
```

And you can use the `kubeconfig` to see what's running on the cluster:

```shell
KUBECONFIG="my-openstack-clusterdeployment1.kubeconfig" kubectl get pods -A
```

## List child clusters

To verify the presence of the child cluster, list the available `ClusterDeployment` objects:

```shell
kubectl get ClusterDeployments -A
```

You'll see output something like this:

```console
NAMESPACE    NAME                              READY   STATUS
kcm-system   my-openstack-clusterdeployment1   True    ClusterDeployment is ready
```

## Tear down the child cluster

To tear down the child cluster, delete the `ClusterDeployment`:

```shell
kubectl delete ClusterDeployment my-openstack-clusterdeployment1 -n kcm-system
```

You'll see confirmation like this:

```console
clusterdeployment.k0rdent.mirantis.com "my-openstack-clusterdeployment1" deleted
```

## Next Steps

Now that you've finished the k0rdent QuickStart, we have some suggestions for what to do next:

Check out the [Administrator Guide](../admin/index.md) ...

* For a more detailed view of k0rdent setup for production
* For details about using k0rdent with cloud Kubernetes distros: AWS EKS and Azure AKS
* For more details about setting up k0rdent to manage clusters on Azure, AWS, VMware and OpenStack

Or check out the [Demos Repository](https://github.com/k0rdent/demos) for fast, makefile-driven demos of k0rdent's key features.
