# K8S Vault tutorial

This repo contains a tutorial on how-to setup hashicorp vault injector and multiple k8s/openshift clusters with an external hashicorp vault

## Prerequisites

### Linux
You will need a laptop/pc/server with Linux and kvm/libvirt. I created this tutorial with Fedora.

### Dnsmasq with networkmanager
If you use the dnsmasq plugin with NetworkManager, see https://fedoramagazine.org/using-the-networkmanagers-dnsmasq-plugin/
Make sure that ```/etc/nsswitch.conf``` is configure correctly, this must be set to ```hosts:       dns files```, and disable ```systemd-resolved``` with ```sudo systemctl is-disabled systemd-resolved``` and reboot you system. 

Make sure dns entries are there, see the following example:

```/etc/hosts```:

```
10.0.0.200 vault.lab.local vault
```

### K8S cluster
We need a k8s cluster or Openshift CRC instance up and running

### Vault server
Have a vault server up and running and also a internal pki

### Have the hashicorp helm chart
See the following doc https://developer.hashicorp.com/vault/docs/platform/k8s/helm

### Firewall rules with Openshift CRC
If you have Openshift CRC up and running you will need to the following firewall rules:

```
sudo firewall-cmd --direct --passthrough ipv4 -I FORWARD -i virbr1 -j ACCEPT
sudo firewall-cmd --direct --passthrough ipv4 -I FORWARD -o virbr1 -j ACCEPT
sudo firewall-cmd --direct --passthrough ipv4 -I FORWARD -i crc -j ACCEPT
sudo firewall-cmd --direct --passthrough ipv4 -I FORWARD -o crc -j ACCEPT
```

If you bridge interface for the vault server is different use a another bridge interface!


## Injector Install
See the following upstream [documentation](https://developer.hashicorp.com/vault/tutorials/kubernetes/kubernetes-external-vault)

### Create override file for injector installation on k8s/openshift (crc)

Create override file ```injector-values.yaml```, see the [example file](helm/injector-values.yaml)

```
global:
  openshift: true
  externalVaultAddr: "https://vault.lab.local:8200"

injector:
  enabled: true
  authPath: "auth/crc"
```

Pay attention to the ```authPath```, we are going to enable authentication for this cluster on it's own path!
In the example from above it is ```crc```, because i am using Openshift CodeReadyContainers.

### Install the vault injector agents

```
$ helm install vault hashicorp/vault -f injector-values.yaml --namespace=vault --create-namespace
```


## Configure authentication
See the following upstream [documentation](https://developer.hashicorp.com/vault/tutorials/kubernetes/kubernetes-external-vault)

Follow the instructions in the documentation, but pay attention to the following changes we do

### Enable kubernetes authentication on specific path

Enable kubernetes authentication, but notice the ```--path=crc```. For every cluster we are going to use a different path, since i am using Openshift CodeReadyContainers, i will put it on path ```crc```


```
$ vault auth enable --path=crc kubernetes

$ vault auth list
Path      Type          Accessor                    Description                Version
----      ----          --------                    -----------                -------
crc/      kubernetes    auth_kubernetes_f9ee1c6f    n/a                        n/a
```

### Configure authentication for the crc openshift instance

When configuring authentication, which basically uses the token from ```serviceAccount``` ```vault``` in namespace ```vault``` for the vault instance to communicate with the K8S cluster. Pay attention to the path, which is ```auth/crc/config```

```
$ vault write auth/crc/config token_reviewer_jwt="$TOKEN_REVIEW_JWT" kubernetes_host="$KUBE_HOST" kubernetes_ca_cert="$KUBE_CA_CERT" issuer="https://kubernetes.default.svc.cluster.local"
```

## Create secret store on the Vault instance

Create a simple keyvalue secret store

```
$ vault secrets enable -path expense/static -version=2 kv
```

### Create kv pair

```
$ MYSQL_DB_PASSWORD=mysuperpassword

$ vault kv put expense/static/mysql db_login_password=${MYSQL_DB_PASSWORD}
```

### Verify the kv pair

```
$ vault kv get expense/static/mysql
====== Secret Path ======
expense/static/data/mysql

======= Metadata =======
Key                Value
---                -----
created_time       2023-01-04T16:18:30.570684018Z
custom_metadata    <nil>
deletion_time      n/a
destroyed          false
version            1

========== Data ==========
Key                  Value
---                  -----
db_login_password    mysuperpassword
```


###  Create policy

We need a policy to allow access to the secret

Create policy file ```policy.hcl```, with the following content

```
path "expense/static/data/mysql" {
  capabilities = ["read", "list"]
}
```

Apply policy
```
$ cat policy.hcl | vault policy write expense-db-mysql -
```

### Assign k8s serviceaccount to role

We now need to assign the policy we just created to serviceaccount ```expense-db-mysql``` in namespace ```expenses```.

```
$ vault write auth/crc/role/expense-db-mysql bound_service_account_names=expense-db-mysql bound_service_account_namespaces=expenses policies=expense-db-mysql ttl=1h
```


## Demo

### Create project

```
$ oc new-project expenses
```
### Create secret from internal pki rootca
Get rootca which is used to sign the certificate for the vault instance

```
$ oc -n expenses create secret generic vault-ca --from-file=ca-cert.pem
```

### Download manifest
This [repo](https://github.com/joatmon08/vault-argocd) contains great examples, so i borrowed the deployment example from there.

```
$ wget https://raw.githubusercontent.com/joatmon08/vault-argocd/part-1/database/deployment.yaml
```

### Add annotations to the ```deployment.yaml``` for custom pki
For a complete list of annotations, see the following [documentation](https://developer.hashicorp.com/vault/docs/platform/k8s/injector/annotations)

We add the following annotations to let the injector agent use the proper rootca to make a connection to the vault instance

```
vault.hashicorp.com/tls-secret: "vault-ca"
vault.hashicorp.com/ca-cert: "/vault/tls/ca-cert.pem"
```

See the modified version [here](deployment/deployment.yaml)

### Deploy the application

```
$ oc apply -f deployment.yaml 
service/expense-db-mysql created
serviceaccount/expense-db-mysql created
deployment.apps/expense-db-mysql created
```

### Verify deployment

```
$ oc get pods
NAME                               READY   STATUS    RESTARTS   AGE
expense-db-mysql-7d788775d-hq45j   2/2     Running   0          32s
```


## Debugging

If the pod does not come up or stays in init stage, check the logs of the sidecar init container.
See the following command as an example

```
$ oc logs expense-db-mysql-7d788775d-hq45j -c vault-agent-init
``` 
