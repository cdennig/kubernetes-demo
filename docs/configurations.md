# Passing secrets and configurations to Pods

As container should be immutable you might to pass information to so you can have different configurations and secrets across different deployments (dev, test, staging, production). Typical examples are database connection strings, API keys, certificates, feature flags or configuration files. In this section we will explore what options are available in Kubernetes.

# Table of Contents
- [Passing secrets and configurations to Pods](#passing-secrets-and-configurations-to-pods)
- [Table of Contents](#table-of-contents)
- [Environmental variables](#environmental-variables)
- [ConfigMaps](#configmaps)
- [Secrets](#secrets)
- [External options](#external-options)
    - [Azure KeyVault](#azure-keyvault)
    - [External Etcd or Consul](#external-etcd-or-consul)
- [Cleanup](#cleanup)

# Environmental variables
Easiest way to pass configuration data to your Pods is by using environmental variables. There are downsides with that approach, but it is easy to start with.

Deploy Pod with environmental variables.
```
kubectl apply -f podEnv.yaml
```

Check env is visible inside your container.
```
kubectl exec env-pod -- env
```

There are few issues with this approach:
* Configuration is part of Pod definition therefore it can reveal sensitive information. As deployment object should be stored in version control this is not ideal.
* As configuration is bounded to Pod definition we cannot split roles of someone managing configurations while other team deploying Pods. We cannot distinguish between the two for example in RBAC.
* Environmental variables are loaded during container start and changes cannot be reflected without restart.
* Complex constructs are not working well with environmental variables. For example complex configuration files cannot be passed that way. You can pass individual keys and construct configuration file in your container manually, but that is inconviniet.
* Values are stored in Kubernetes system unencrypted so if attacker would gain access to underlaying Etcd, he can see all passwords

# ConfigMaps
We will solve most of previously outlined issues with ConfigMap. 

First let's deploy ConfigMap with configurations. We will include two key value pairs and one multi-line statement (for example ini config file).

```
kubectl apply -f configMap.yaml
```

In our first example we will create Pod and load all keys from our ConfigMap as environmental variables and check results out.
```
kubectl apply -f podCm1.yaml
kubectl exec cm1-pod -- env
```

We see that simple key:value pairs are OK, but complex configuration file is malformed. Perhaps we should just those simple keys and deal with complex one later. With that we can also rename those key. For example you might ConfigMap with some key:value that is used with multiple different Pods. In those Pods you expect different naming convention for keys. Rather than keeping two ConfigMaps with the same values, we can remap names in Pod definition.
```
kubectl apply -f podCm2.yaml
kubectl exec cm2-pod -- env
```

Now what about more complex value that represents something like configuration file? Another method to pass such information to Pod is leverage Volume and map it as file.
```
kubectl apply -f podCm3.yaml
kubectl exec cm3-pod -- ls /etc/config
kubectl exec cm3-pod -- cat /etc/config/configfile.ini
```

Good thing about accessing data via files is that it gets updated without restart (it might take some time for Kubernetes to reflect this change, can be up to one minute). We will now update config map and test it out. While value in env will remain unchanged, value in file will reflect change.
```
kubectl apply -f ConfigMap2.yaml
kubectl exec cm3-pod -- env | grep newnamekey1
kubectl exec cm3-pod -- cat /etc/config/mycfgkey
```

Sometimes it is not convenient to maintain complex configuration files in ConfigMap yaml. You can also use kubectl and point it to specific file or directory and create ConfigMap that way. Note however that this is not declarative way.
```
kubectl create configmap newmap --from-file=./configs/
kubectl describe cm newmap
```

When you need to deploy a lot of ConfigMaps, Deployments, Services and Secrets and would like to have some variables that change for each deployment consider packaging everything up with [Helm](docs/helm.md)

# Secrets
Currenlty secrets behave pretty much the same as ConfigMaps, but there is difference in how it is implemented internally. Secrets are going to be stored in encrypted way in Kubernetes Etcd and there is option being prepared to use Azure KeyVault to protect encryption keys with specialized system (envelope technique). Since Secrets and ConfigMaps are different object you can apply different RBAC to them. For example one user can be responsible for maitaining ConfigMaps, but someone else is managing Secrets.

From operations perspective only major differenc when using Secrets is that they are store in base64 encoding. Note this is not for confidentiality (consider this plain text), rather to prevent some coding issues with some characters.

Let's deploy some secret.
```
kubectl apply -f secret.yaml
```

Note that password is base64 encoded so it actually stands for this:
```
echo -n 'QXp1cmUxMjM0NTY3OA==' | base64 -d
Azure12345678
```

If you create Secret in imperative way from file kubectl will do base64 encoding automatically.
```
echo -n 'Azure87654321' > ./password.txt
kubectl create secret generic secret2 --from-file=./password.txt 
rm ./password.txt
```

We can use Secret in Pod in similar way to ConfigMaps either via env or files.
```
kubectl apply -f podSecret.yaml
kubectl exec sec -- env | grep SECRET
```

# External options
There might be reasons not to bound any of that with Kubernetes system. Maybe you have very high standards for storing and managing secrets in dedicated hardware-supported (HSM) solutions like Azure KeyVault. Or you have complex configurations you want to centralize and make available not only for apps running in single Kubernetes cluster, but many clusters, Azure Container Instances, VMs or PaaS. In that case you might consider deploying centralized configuration store such as Etcd or Consul.

## Azure KeyVault

TBD

## External Etcd or Consul

TBD

# Cleanup

```
kubectl delete -f podEnv.yaml
kubectl delete -f podCm1.yaml
kubectl delete -f podCm2.yaml
kubectl delete -f podCm3.yaml
kubectl delete -f podSecret.yaml
kubectl delete -f secret.yaml
kubectl delete secret secret2
kubectl delete cm newmap
kubectl delete -f configMap.yaml
```