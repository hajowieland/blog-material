# Dive into Cluster API, ClusterClass and CAPH

The resources mentioned in this [blog post](https://wieland.tech/blog/cluster-api-on-hetzner-with-clusterclass).


## IMPORTANT

The resources in this repo do not work out of the box.

### Hetzner SSH Key

You also need to a patch to replace the Hetzner SSH Key in the [HetznerClusterTemplate](resources/hetznerclustertemplate.yaml).

See the [Kustomization with path](#kustomization-with-patch) section below.


### EncryptionConfiguration

You need to deploy a Secret with your `EncryptionConfiguration` and your Hetzner Cloud token first. Check the blog post for more details.


```shell
openssl rand -base64 32
```


```yaml showLineNumbers
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
      - secrets
    providers:
      - secretbox:
          keys:
            - name: key1
              secret: my-super-secret-changethisplz
      - identity: {}
```

```shell
kubectl create secret generic encryption-config \
  --from-file=encryptionconfig.yaml=encryption-provider.yaml
```


### Hetzner Cloud Token

```shell
# keep trailing space to not expose token in your shell history
 export HCLOUD_TOKEN="your-hcloud-token"

kubectl create secret generic hetzner \
  --from-literal=hcloud=$HCLOUD_TOKEN
```

## Kustomization with patches

If you have created both Secrets, you just need to adapt the patch to match the Hetzner SSH Key name for your project. Then we patch the kubernetes version and set the image name you created in [Part1](https://wieland.tech/blog/cluster-api-image-builder-hcloud) for your Cluster.

```yaml showLineNumbers
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: hajowieland-blog-material
spec:
  interval: 5m
  url: https://github.com/hajowieland/blog-material
  ref:
    branch: main
---
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: test-cluster
spec:
  interval: 10m
  patches:
  - patch: |-
      apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
      kind: HetznerClusterTemplate
      metadata:
        name: hetzner-0.1.0
      spec:
        template:
          spec:
            sshKeys:
              hcloud:
                - name: your-hetzner-ssh-key-name # replace me
  - patch: |-
      apiVersion: cluster.x-k8s.io/v1beta1
      kind: Cluster
      metadata:
        name: test
      spec:
        topology:
          variables:
            - name: imageName
              value: cluster-api-flatcar-stable-4152.2.2-v1.32.4-1745487823 # replace me
            - name: instanceType
              value: cx22
          version: v1.32.4 # replace me
          controlPlane:
            variables:
              overrides:
                - name: imageName
                  value: cluster-api-flatcar-stable-4152.2.2-v1.32.4-1745487823 # replace me
                - name: instanceType
                  value: cx22
  sourceRef:
    kind: GitRepository
    name: hajowieland-blog-material
  path: "./cluster-api-on-hetzner-with-clusterclass/resources"
  prune: true
  timeout: 1m
```
