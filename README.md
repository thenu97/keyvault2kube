# Copies secrets from KeyVault into Kubernetes

## Deployment

Ideally the container should get KeyVault credentials from a managed service identity using something like the 
`aad-pod-identity` project but it will also respect the env vars of `AZURE_CLIENT_ID`, `AZURE_CLIENT_SECRET` and `AZURE_TENANT_ID`.

To configure which KeyVaults to scan, an env var of `KEY_VAULT_URLS` with a comma separated list of Key Vault URL's is all thats needed. 

Deployment:
```
kubectl apply -f https://TODO
```

Example deployment.yaml mounting a config map to be used as a template (this example is missing the RBAC roles so check `deployment.yaml` for those):
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: keyvault2kube
  namespace: kube-system
  labels:
    app: keyvault2kube
spec:
  replicas: 1
  selector:
    matchLabels:
      app: keyvault2kube
  template:
    metadata:
      labels:
        app: keyvault2kube
    spec:
      containers:
        - name: keyvault2kube
          image: terrycain/keyvault2kube:latest
          volumeMounts:
            - name: config-volume
              mountPath: /app/templates/
      volumes:
        - name: config-volume
          configMap:
            name: keyvault2kube-template
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: keyvault2kube-template
  namespace: kube-system
data:
  template1.yaml: |
    test1: "hello {data}"
    test2: "{data2} world"
```

## Secret Configuration

Keyvault Secret tags control how and what secrets are copied into keyvault:
```
k8s_secret_name | required | Name of the secret in Kubernetes 
k8s_secret_key  | optional | Name of the key to store the value against. If the secret content type is not `application/json` or `text/x-yaml` then this is required
k8s_namespaces  | optional | Comma separated list of namespaces to apply the secret to, otherwise default is used
k8s_type        | optional | Either Opaque or `kubernetes.io/dockerconfigjson`
k8s_convert     | optional | Look below
```

If the `k8s_secret_name` is applied to multiple KeyVault secrets and the `k8s_namespaces` are the same and the `k8s_secret_key` tags 
are different, the secret will be combined into 1 Kubernetes secret with multiple keys.

If a content type is applied to the secret and is `application/json` or `text/x-yaml`, the value will be decoded and used as the content for 
the secret. E.g. if a secret contained `{"a": "1", "b": "2"}` then the Kubernetes secret would have 2 keys, "a" and "b".

If `k8s_convert` is added to a secret with a value of `dockerconfigjson`, a content type of `application/json`, and a 
json value with the fields "registry", "email", "username", "password" are provided then it'll convert that json 
into the dockerconfigjson format and secret type.

-- TODO will change this to use Jinja2 instead before V1
If `k8s_convert` has a value like `file:/app/templates/template1.yaml` (must end with .yaml), the secret is a json document and has a content type of `application/json`, then
the yaml file will be read in, python's .format(**json_document) will be templated over the yaml document and then loaded 
and used as the secret data. E.g. a secret value of `{"data": "world", "data2": "hello"}`
```yaml
test1: "hello {data}"
test2: "{data2} world"
```
would result in:
```yaml
data:
  test1: hello world
  test2: hello world
```
