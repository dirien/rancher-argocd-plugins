# How To create an Argo CD Plugin? 

## Motivation

The recent tweet from the argoproj, was the motivation to get my hands "dirty" and play a little with `kbld`. But I install thought about try to integrate `kbld` with the Argo CD management plugin system. Always wanted to do something with this interesting looking functionality.

%[https://twitter.com/argoproj/status/1517201502246301696?s=20&t=4fLiid2JaBZnFPapSvIFYQ]

## About kbld

[kbld](https://carvel.dev/kbld/docs/v0.33.0/) is a tool for building Docker images and resolving image references. We are not using in this tutorial the building part of `kbld`. Instead, we focus on the image references resolving part.

When you apply `kbld` an on existing manifest, you will see that image digest reference (e.g.
index.docker.io/your-username/your-repo@sha256:
4c8b96...) was used inserted instead of a tagged reference (e.g. kbld:docker-io...).

Because tags are mutable, they have a certain disadvantages when you use them to deploy an image:

In Kubernetes, deploying by tag can result in unexpected behaviours. For example, assume that you have an existing `Deployment` resource that references a container image by tag 0.0.1. To fix a bug or make a small change, your build process creates a new image but with the same tag 0.0.1

New `Pods` that are created from your Deployment resource can end up using either the old or the new image, even if you don't change your Deployment resource specification.

Digest references are preferred to other image reference forms as they are immutable, hence provide a guarantee that exact version of built software will be deployed.

## What Are Argo CD Plugins?

Argo CD allows us to integrate more config management tools using config management plugins. Most prominent inbuilt tools are helm and kustomize.

We have to option, when it comes to create our own `Config Management Plugin` (CMP):

1. Add the plugin config to the main Argo CD ConfigMap. Our repo-server container will then run the plugin's commands.

According tho the official documentation of Argo CD this is a good option for a simple plugin that requires only a few lines of code. As it would still nicely fit into the Argo CD ConfigMap.

2. Add the plugin as a sidecar to the repo-server Pod.

This option is the way to go for more complex plugins, which would bloat our Argo CD ConfigMap.

## How To Crate The kbld Plugin?

In this blog, I am going to use the option 1. And I will use the helm chart to deploy Argo CD.

Tha means I going to to add the following lines to my `values.yaml` file:

```yaml
  config:
    configManagementPlugins: |
      - name: kbld
        generate:                 
          command: ["bash", "-c"]
          args: ['helm template --release-name "$ARGOCD_APP_NAME" -f <(echo "$HELM_VALUES") . > kbld.yaml && kbld -f kbld.yaml >> final.yaml && cat final.yaml']
```
Keep in mind, that a plug consist of the commands: `init` and `generate`. The `init` command is optional and takes care to initialize application source directory.

The `generate` command must print a valid YAML or JSON stream to stdout.

If you need both, your definition would look like this:

```yaml
...
    - name: pluginName
      init:                        
        command: ["sample command"]
        args: ["sample args"]
      generate:                     
        command: ["sample command"]
        args: ["sample args"]
...
```

To pass any helm values, I created the `HELM_VALUES` environment variable. The other variable `ARGOCD_APP_NAME` is one of the default [environment variables](https://argo-cd.readthedocs.io/en/stable/user-guide/build-environment/) of Argo CD.

The second part we need now to do now, to get this plugin to work is to download the `kbld` binary and add it to the `argocd_repo_server`

For this, we're going to use an init container and a shared volume.

The init container will download the `kbld` binary and save it to shared volume. The shared volume will be then mounted to the `argocd_repo_server` container.

```yaml
repoServer:
  initContainers:
    - name: download-tools
      image: busybox:1.35.0
      command: [ sh, -c ]
      env:
        - name: KBLD_VERSION
          value: "0.33.0"
      args:
        - wget -q https://github.com/vmware-tanzu/carvel-kbld/releases/download/v${KBLD_VERSION}/kbld-linux-amd64 &&
          mv kbld-linux-amd64 /custom-tools/kbld &&
          chmod +x /custom-tools/kbld
      volumeMounts:
        - mountPath: /custom-tools
          name: custom-tools
  volumes:
    - name: custom-tools
      emptyDir: { }
  volumeMounts:
    - mountPath: /usr/local/bin/kbld
      name: custom-tools
      subPath: kbld
```

If you prefer to use option 2, providing the CMP as a sidecar to the repo-server Pod check out the [Argo CD docs](https://argo-cd.readthedocs.io/en/stable/user-guide/config-management-plugins/#option-2-configure-plugin-via-sidecar)

## Rancher Desktop

For this example, I am going to use the [Rancher Desktop](https://rancherdesktop.io/) for my local Kubernetes cluster.

Depending on your operating system, you may have a different way of installing Rancher Desktop. Check the installation guide on their website. -> https://docs.rancherdesktop.io/getting-started/installation

## Install The ArgoCD Helm chart

I put both snippets into a values file (`argocd-values.yaml`) and installed the ArgoCD Helm chart.

```bash
helm upgrade -i  my-argo-cd argo/argo-cd --version 4.5.4 -f argocd-values.yaml
```

## Test The Plugin

So let's test the plugin. All we need to do is to create a new application in Argo CD. The application manifest will look like this:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: test
  namespace: default
spec:
  destination:
    namespace: default
    server: https://kubernetes.default.svc
  project: default
  source:
    chart: minecraft-exporter
    plugin:
      env:
        - name: HELM_VALUES
          value: |
            replicaCount: 1
      name: kbld
    repoURL: https://dirien.github.io/minecraft-prometheus-exporter
    targetRevision: 0.5.0
```

Take note of the new `plugin` section. We are using here the name of the plugin we created earlier. And as mentioned before, we are using the `HELM_VALUES` environment variable to pass any helm values.

If everything is working, you should see  all the images tags are replaced with their image digest reference.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  annotations:
    kbld.k14s.io/images: |
      - origins:
        - resolved:
            tag: 0.13.0
            url: ghcr.io/dirien/minecraft-exporter:0.13.0
        url: ghcr.io/dirien/minecraft-exporter@sha256:4a6a3acd7fdcdf7f93817dcba54b2928aa02b33b132b44ddcc608f693f8e8723
  labels:
    app: minecraft-exporter
...
spec:
  containers:
    - image: >-
        ghcr.io/dirien/minecraft-exporter@sha256:4a6a3acd7fdcdf7f93817dcba54b2928aa02b33b132b44ddcc608f693f8e8723
      imagePullPolicy: IfNotPresent
...
```

## Wrap Up

This is just one little example, on how to use the Argo CD CMP functionality. And its actually a quite good example, because we avoid using the tag reference, but instead use the digest reference.

Feel free to check the Argo CD docs for more information about the [usage of plugins](https://argo-cd.readthedocs.io/en/stable/user-guide/config-management-plugins/#plugins)
