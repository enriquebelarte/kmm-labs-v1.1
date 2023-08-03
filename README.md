# KMM v1.1 Labs

KMM Labs is a series of examples and use cases of Kernel Module Management Operator running on an Openshift cluster.

## Installation

In order to know more about what KMM is and how to install it you can refer to the [documentation](https://docs.openshift.com/container-platform/4.13/hardware_enablement/kmm-kernel-module-management.html) site.

## Driver Toolkit

We could use any image that includes the necessary libraries and dependencies for building a kernel module for our specific version. However, for ease, we can leverage Driver Toolkit (DTK), which is a container image used as a base image for building kernel modules.

When using DTK, we should set which version of the Driver Toolkit [image](https://github.com/openshift/driver-toolkit#finding-the-driver-toolkit-image-url-in-the-payload) we are using to match the exact kernel version running on OpenShift nodes. In our case, we are using DTK with KMM and KMM will automatically add the appropriate ${DTK_AUTO} value.

We can get further info at DTK [repository](https://github.com/openshift/driver-toolkit#readme).


## In-Cluster build module from sources

In this example we will build the kernel module in our cluster from a sources in a git repository.
First we will add a `ConfigMap` that will contain all the information about the build process such as the image we are using (DTK in our case),
the remote git repository where sources are hosted or the compiling commands.
After that we will create the Module object itself which will set the registry where the resulting image will be, the secret for a possible authentication, the previous `ConfigMap` that will be used or which nodes should the pods be scheduled on.

Applying next file in an OpenShift cluster should build and load a kmm-ci-a module in the nodes labeled with a `worker` role:

```yaml
    ---
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: build-module-labs
      namespace: openshift-kmm
    data:
      dockerfile: |
        ARG DTK_AUTO
        ARG KERNEL_VERSION
        FROM ${DTK_AUTO} as builder
        WORKDIR /build/
        RUN git clone -b main --single-branch https://github.com/rh-ecosystem-edge/kernel-module-management.git
        WORKDIR kernel-module-management/ci/kmm-kmod/
        RUN make
        FROM docker.io/redhat/ubi8-minimal
        ARG KERNEL_VERSION
        RUN microdnf -y install kmod openssl && \
            microdnf clean all && \
            rm -rf /var/cache/yum
        RUN mkdir -p /opt/lib/modules/${KERNEL_VERSION}
        COPY --from=builder /build/kernel-module-management/ci/kmm-kmod/*.ko /opt/lib/modules/${KERNEL_VERSION}/
        RUN /usr/sbin/depmod -b /opt
    ---
    apiVersion: kmm.sigs.x-k8s.io/v1beta1
    kind: Module
    metadata:
      name: kmm-ci-a
      namespace: openshift-kmm
    spec:
      moduleLoader:
        container:
          modprobe:
            moduleName: kmm-ci-a
          kernelMappings:
            - regexp: '^.*\.x86_64$'
              containerImage: quay.io/myrepo/kmmo-lab:kmm-kmod-${KERNEL_FULL_VERSION}
              build:
                dockerfileConfigMap:
                  name: build-module-labs
      imageRepoSecret: 
        name: myrepo-pull-secret
      selector:
        node-role.kubernetes.io/worker: ""

```

## In-Cluster build module from sources and sign it

Signed kernel modules provide a mechanism for the kernel to verify the integrity of a module primarily for use with UEFI Secure Boot.


This next example is pretty much the same as above except that we will generate a private key and certificate to sign the resulting .ko file after the build process.


Generate key and certificate files and create secrets based on them:
```bash
     openssl req -x509 -sha256 -nodes -days 365 -newkey rsa:2048 -keyout private.key -out certificate.crt
     oc create secret generic mykey --from-file=key=private.key
     oc create secret generic mycert --from-file=cert=certificate.crt
```

Build and sign resulting module file:
```yaml
    ---
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: labs-dockerfile
      namespace: openshift-kmm
    data:
      dockerfile: |
        ARG DTK_AUTO
        ARG KERNEL_VERSION
        FROM ${DTK_AUTO} as builder
        WORKDIR /build/
        RUN git clone -b main --single-branch https://github.com/rh-ecosystem-edge/kernel-module-management.git
        WORKDIR kernel-module-management/ci/kmm-kmod/
        RUN make
        FROM docker.io/redhat/ubi8-minimal
        ARG KERNEL_VERSION
        RUN microdnf -y install kmod openssl && \
            microdnf clean all && \
            rm -rf /var/cache/yum
        RUN mkdir -p /opt/lib/modules/${KERNEL_VERSION}
        COPY --from=builder /build/kernel-module-management/ci/kmm-kmod/*.ko /opt/lib/modules/${KERNEL_VERSION}/
        RUN /usr/sbin/depmod -b /opt
    ---
    apiVersion: kmm.sigs.x-k8s.io/v1beta1
    kind: Module
    metadata:
      name: labs-signed-module
      namespace: openshift-kmm
    spec:
      moduleLoader:
        container:
          modprobe:
            moduleName: kmm_ci_a
          kernelMappings:
            - regexp: '^.*\.x86_64$'
              containerImage: quay.io/myrepo/kmmo-lab:signed-module-1.0
              build:
                dockerfileConfigMap:
                  name: labs-dockerfile
              sign:
                keySecret:
                  name: mykey
                certSecret:
                  name: mycert
                filesToSign:
                  - /opt/lib/modules/$KERNEL_FULL_VERSION/kmm_ci_a.ko
      imageRepoSecret: 
        name: myrepo-pull-secret
      selector: 
        node-role.kubernetes.io/worker: ""
```

## Load an existing module
This is the simplest example in the Labs. We have an existing image of the kernel module for a specific kernel version and we want KMM to manage and load it:
```yaml
    ---
    apiVersion: kmm.sigs.x-k8s.io/v1beta1
    kind: Module
    metadata:
      name: kmm-ci-a
      namespace: openshift-kmm
    spec:
      moduleLoader:
        container:
          modprobe:
            moduleName: kmm-ci-a
          kernelMappings:
            - literal: '4.18.0-372.52.1.el8_6.x86_64'
              containerImage: quay.io/myrepo/kmmo-lab:kmm-kmod-4.18.0-372.52.1.el8_6.x86_64
      imageRepoSecret: 
        name: myrepo-pull-secret
      selector:
        node-role.kubernetes.io/worker: ""
```

## Remove an in-tree module before loading another one

We might need to remove an existing in-tree module before loading an out-of-tree one.
Some use cases may be including the addition of more features, fixes, or patches to the out-of-tree module for the same driver, as well as encountering incompatibilities between the in-tree and out-of-tree modules.

To achieve this we can remove in-tree modules just adding `inTreeModuleToRemove: <NameoftheModule>`. In our example we wil remove `joydev` module which is the standard in-tree driver for joysticks and similar devices:

```yaml
    ---
    apiVersion: kmm.sigs.x-k8s.io/v1beta1
    kind: Module
    metadata:
      name: kmm-ci-a
      namespace: openshift-kmm
    spec:
      moduleLoader:
        container:
          modprobe:
            moduleName: kmm-ci-a
          kernelMappings:
            - literal: '4.18.0-372.52.1.el8_6.x86_64'
              containerImage: quay.io/myrepo/kmmo-lab:kmm-kmod-4.18.0-372.52.1.el8_6.x86_64
          inTreeModuleToRemove: joydev
      imageRepoSecret: 
        name: myrepo-pull-secret
      selector:
        node-role.kubernetes.io/worker: ""
```

## Device Plugin

We will use an existing plugin to simulate device plugins called  [K8S-dummy-device-plugin](https://github.com/redhat-nfvpe/k8s-dummy-device-plugin) 
```bash
$ oc exec -it dummy-pod -- printenv | grep DUMMY_DEVICES
DUMMY_DEVICES=dev_3,dev_4
```
```yaml
        ---
        apiVersion: kmm.sigs.x-k8s.io/v1beta1
        kind: Module
        metadata:
          name: kmm-ci-a
          namespace: openshift-kmm
        spec:
          devicePlugin:
            container:
              image: "quay.io/myrepo/oc-dummy-device-plugin:1.1"
          moduleLoader:
            container:
              modprobe:
                moduleName: kmm-ci-a
              kernelMappings:
                - literal: '4.18.0-372.52.1.el8_6.x86_64'
                  containerImage: quay.io/myrepo/kmmo-lab:kmm-kmod-4.18.0-372.52.1.el8_6.x86_64
          imageRepoSecret:·
            name: myrepo-pull-secret
    ····
          selector:
            node-role.kubernetes.io/worker: ""
```

##  Ordered upgrade of kernel module without reboot

Whenever we need to upgrade a kernel module we should do it node by node so the impact on the workload running in the cluster.
We should first label the specific node in this format:

```
kmm.node.kubernetes.io/version-module.<module-namespace>.<module-name>=$moduleVersion
```

In our case we'll use this example:
```bash
    oc label node my-node-1 kmm.node.kubernetes.io/version-module.openshift-kmm.kmm-ci-a=1.0
```

Then we can use a new build based on previous build example just adapting the ModuleLoader and reusing existing Configmap for build. Make sure you have deleted any  Module object left from past examples:

```yaml {}[build_v1.yaml]
    ---
    apiVersion: kmm.sigs.x-k8s.io/v1beta1
    kind: Module
    metadata:
      name: kmm-ci-a
      namespace: openshift-kmm
    spec:
      moduleLoader:
        container:
          version: "1.0"
          modprobe:
            moduleName: kmm-ci-a
          kernelMappings:
            - regexp: '^.*\.x86_64$'
              containerImage: quay.io/myrepo/kmmo-lab:kmm-kmod-${KERNEL_FULL_VERSION}-v1.0
              build:
                dockerfileConfigMap:
                  name: build-module-labs
      imageRepoSecret: 
        name: myrepo-pull-secret
      selector:
        node-role.kubernetes.io/worker: ""
```


## Loading soft dependencies

Kernel modules sometimes have dependencies on others modules so those dependencies have to be loaded in advance. In the following example `kmm-ci-a` depends on `kmm-ci-b` so we will set `modulesLoadingOrder` and then the list with the `moduleName` as the first entry followed by all of its dependencies. In our case that is just `kmm-ci-b` but it could be a longer list.

```yaml
    ---
    apiVersion: kmm.sigs.x-k8s.io/v1beta1
    kind: Module
    metadata:
      name: kmm-ci-a
      namespace: openshift-kmm
    spec:
      moduleLoader:
        container:
          modprobe:
            moduleName: kmm-ci-a
            modulesLoadingOrder:
              - kmm-ci-a
              - kmm-ci-b
          kernelMappings:
            - regexp: '^.*\.x86_64$'
              containerImage: quay.io/ebelarte/kmmo-lab:kmm-kmod-${KERNEL_FULL_VERSION}
      imageRepoSecret: 
        name: ebelarte-pull-secret
      selector:
        node-role.kubernetes.io/worker: ""
```
## Troubleshoot

