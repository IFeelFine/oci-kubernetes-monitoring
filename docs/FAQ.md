## FAQ

### Can I use kubectl do deploy the solution?

Helm is the recommended method of deployment. kubectl based deployment can be done by generating individual templates using helm. Refer [this](README.md#kubectl) for details.

### Can I use my own ServiceAccount ?

**Note**: This is supported only through the helm chart based deployment.

By default, a cluster role, cluster role binding and serviceaccount will be created for the Fluentd and Management Agent pods to access (readonly) various Kubernetes Objects within the cluster for supporting logs, objects and metrics collection. However, if you want to use your own serviceaccount, you can do the same by setting the "oci-onm-common.createServiceAccount" variable to false and providing your own serviceaccount in the "oci-onm-common.serviceAccount" variable. Ensure that the serviceaccount should be in the same namespace as the namespace used for the whole deployment. The namespace for the whole deployment can be set using the "oci-onm-common.namespace" variable, whose default value is "oci-onm".

The serviceaccount must be binded to a cluster role defined in your cluster, which allows access to various objects metadata. The following sample is a recommended minimalistic role definition as of chart version 3.0.0.

```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: oci-onm
rules:
  - apiGroups:
      - ""
    resources:
      - '*'
    verbs:
      - get
      - list
      - watch
  - nonResourceURLs: ["/metrics"]
    verbs: ["get"]
  - apiGroups:
      - apps
      - batch
      - discovery.k8s.io
      - metrics.k8s.io
    resources:
      - '*'
    verbs:
      - get
      - list
      - watch
```

Once you have the cluster role defined, to bind the cluster role to your serviceaccount use the following cluster role binding definition.

```
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: oci-onm
roleRef:
  kind: ClusterRole
  name: oci-onm
  apiGroup: rbac.authorization.k8s.io
subjects:
  - kind: ServiceAccount
    name: <ServiceAccountName>
    namespace: <Namespace>
```

### How to set encoding for logs ?

**Note**: This is supported only through the helm chart based deployment.

By default Fluentd tail plugin that is being used to collect various logs has default encoding set to ASCII-8BIT. To overrided the default encoding, use one of the following approaches.

#### Global level

Set value for encoding under fluentd:tailPlugin section of values.yaml, which applies to all the logs being collected from the cluster.

```
..
..
oci-onm-logan:
  ..
  ..
  fluentd:
    ...
    ...
    tailPlugin:
      ...
      ...
      encoding: <ENCODING-VALUE>
```

#### Specific log type level

The encoding can be set at invidivual log types like kubernetesSystem, linuxSystem, genericContainerLogs, which applies to all the logs under the specific log type.

```
..
..
oci-onm-logan:
  ..
  ..
  fluentd:
    ...
    ...
    kubernetesSystem:
      ...
      ...
      encoding: <ENCODING-VALUE>
```

```
..
..
oci-onm-logan:
  ..
  ..
  fluentd:
    ...
    ...
    genericContainerLogs:
      ...
      ...
      encoding: <ENCODING-VALUE>
```

#### Specific log level

The encoding can be set at individual log level too, which takes precedence over all others.

```
..
..
oci-onm-logan:
  ..
  ..
  fluentd:
    ...
    ...
    kubernetesSystem:
      ...
      ...
      logs:
        kube-proxy:
          encoding: <ENCODING-VALUE>
```

```
..
..
oci-onm-logan:
  ..
  ..
  fluentd:
    ...
    ...
    customLogs:
        custom-log1:
          ...
          ...
          encoding: <ENCODING-VALUE>
```

### How to use Configfile based AuthZ (User Principal) instead of default AuthZ (Instance Principal) ?

**Note**: This is supported only through the helm chart based deployment.

The default AuthZ configuration for connecting to OCI Services from the monitoring pods running in the Kubernetes clusters is `InstancePrincipal` and it is the recommended approach for OKE. If you are trying to monitor Kubernetes clusters other than OKE, you need to use `config` file based AuthZ instead.

First you need to have a OCI local user (preferrably a dedicated user created only for this use-case so that you can restrict the policies accordingly) and OCI user group. Then you need to generate API Signing key and policies. 

  * Refer [OCI API Signing Key](https://docs.oracle.com/en-us/iaas/Content/API/Concepts/apisigningkey.htm) for instructions on how to generate API Signing key for a given user.
  * Refer [this](README.md#pre-requisites) for creating required policies.

#### Helm configuration

Modify your override_values.yaml to add the following. 

```
...
...
oci-onm-logan:
  ...
  ...
  authtype: config
  ## -- OCI API Key Based authentication details. Required when authtype set to config
  oci:
   # -- Path to the OCI API config file
   path: /var/opt/.oci
   # -- Config file name
   file: config
   configFiles:
      config: |-
         # Replace each of the below fields with actual values.
         [DEFAULT]
         user=<user ocid>
         fingerprint=<fingerprint>
         key_file=/var/opt/.oci/private.pem
         tenancy=<tenancy ocid>
         region=<region>
      private.pem: |-
      #   -----BEGIN RSA PRIVATE KEY-----
      #   XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
      #   -----END RSA PRIVATE KEY-----
``` 

### Enable Multi Process Workers

**Note**: This feature will not work with custom fluentd configuration and if you set custom fluentd configuration, it would be placed under worker 0.

To enable multi-process workers feature of Fluentd, set value of `multiProcessWorkers` under oci-onm-logan to the desired number of workers. By default it is set 0, indicating that the feature is disabled. To assign different workers for different logs, you may set `worker` field to the desired worker id against each of the individual logs or group of logs as supported in the confguration. The default worker id is set to 0 for all the logs when enabling `multiProcessWorkers` feature. The following are few examples,
 
#### Example1  

Enable multi-process worker feature with 2 workers and set the worker id 1 to all container logs (except custom logs) and keeping the default worker id (0) for the remaining all logs. 

```      
..       
..       
oci-onm-logan:
  ..  
  ..  
  fluentd:
    ...   
    ...
    multiProcessWorkers: 2
    ...
    ...
    genericContainerLogs:
      ...
      ... 
      worker: 1
```       

#### Example2                 

Enable multi-process worker feature with 3 workers and set the worker id 1 to all container logs (except custom logs), worker id 2 to Kube Proxy, Linux Syslog and Kubelet logs, and keeping the default worker id (0) for the remaining all logs.    

```
..
..
oci-onm-logan:
  ..
  ..
  fluentd:
    ...
    ...
    multiProcessWorkers: 3
    ...
    ...
    kubernetesSystem:
    ...
    ...
      logs:
        ...
        ...
        kube-proxy:
        ...
        ...
        worker: 2
        ...
        ...
    ...
    ...
    linuxSystem:
      ...
      ...
      logs:
        ...
        ...
        syslog:
          ...
          ...
          worker: 2
          ...
          ...
    ...
    ...
    genericContainerLogs:
      ...
      ...
      worker: 1
```       

