# How to install:

## Prerequisites

You will need `kubectl`, `kustomize` and `helm`, the way to install these binaries is out of the scope of this documentation.

## Create boot disk with the Talos factory service

On your personal computer, browse to https://factory.talos.dev/.
Enable secure boot during the wizard, and select the desired system extentions.

```yaml
customization:
  systemExtensions:
    officialExtensions:
      - siderolabs/bnx2-bnx2x
      - siderolabs/i915
      - siderolabs/intel-ucode
```

Copy the URL of the initial installation media (https://factory.talos.dev/image/3443c1e327301535433b31c61d71a4cbb14d8b90c05fac977bf064d98ce331a7/v1.10.5/metal-amd64-secureboot.iso) and the name of the Initial Installation image: `factory.talos.dev/metal-installer-secureboot/3443c1e327301535433b31c61d71a4cbb14d8b90c05fac977bf064d98ce331a7:v1.10.5`.

## Burn initial installation media on usb

On your personal computer, download the ISO and burn it on a usb key to prepare the installation.

```sh
$ curl -qs https://factory.talos.dev/image/3443c1e327301535433b31c61d71a4cbb14d8b90c05fac977bf064d98ce331a7/v1.10.5/metal-amd64-secureboot.iso -o metal-amd64-secureboot.iso
$ dd if=metal-amd64-secureboot.iso of=/dev/sdxxxx #(usb key)
```

## Enroll the secure boot keys from Sidero Labs

Plug the usb key on the target computer, and boot it on the usb key.
During boot, select in the menu the option to enroll the secureboot keys.

## Reboot and boot on talos (in memory), apply the configuration 

Reboot on the usb key on the target computer. Select the first entry in the menu to boot on Talos.
Once the computer is fully started, you should see the Talos Dashboard on the target computer, the network should be "connected" and it shoud be "ready"

Test connectivity with talosctl, starting by installing it:
```sh
$ curl -Ls https://github.com/siderolabs/talos/releases/latest/download/talosctl-linux-amd64 -o talosctl
$ chmod +x talosctl
$ mv taloctl xxxx # move in $PATH
$ talosctl -n IP_OF_NODE get ethernetstatus --insecure
$ talosctl -n IP_OF_NODE get disks --insecure
```

Identify the network interface and model name of the disk where we want to install Talos.

Generate an initial configuration:
```sh
$ talosctl gen secrets -o secrets.yaml 
$ talosctl gen config --with-secrets secrets.yaml myKluster https://192.168.x.x:6443
```

Network interface: enp1s0f1
Hardrive model: TOSHIBA MQ01ABF0

Modify the configuration, starting with network configuration, using the previously identified network interface:
```yaml
machine:
  network:
    hostname: xxxxxx
    interfaces:
      - interface: enp1s0f1
        dhcp: true
        vip:
          ip: 192.168.x.x
    searchDomains:
      - home
```

The install part, we use the previous identified installer image and hardrive model:
```yaml
machine:
  install:
    image: factory.talos.dev/metal-installer-secureboot/3443c1e327301535433b31c61d71a4cbb14d8b90c05fac977bf064d98ce331a7:v1.10.5
      diskSelector:
        model: TOSHIBA MQ01ABF0
```

We allow to schedule pods on the master nodes
```yaml
machine:
  # nodeLabels:
  #   node.kubernetes.io/exclude-from-external-load-balancers: ""
  allowSchedulingOnControlPlanes: true
```

Now, let's configure the local storage, by adding this API definition at the end of the manifest:
```yaml
---
apiVersion: v1alpha1
kind: VolumeConfig
name: EPHEMERAL
provisioning:
  diskSelector:
    match: system_disk
  minSize: 2GiB
  maxSize: 200GiB
  grow: false
```

If we do not do that, the EPHEMERAL volume will use all the remaining space on the disk. We don't want that because we want to create a dedicated space to store the VMs.

Now, let's apply the configuration to the node:
```sh
$ talosctl apply-config --insecure -n 192.168.x.x --file controlplane.yaml
```

# Kubernetes

## Now, we can bootstrap kubernetes
```sh
$ talosctl -n 192.168.x.x -e 192.168.x.x --talosconfig=./talosconfig bootstrap
```

## Generation of the kubeconfig and merge it with the local config

```sh
$ talosctl kubeconfig --nodes 192.168.x.x --endpoints 192.168.x.x --talosconfig=./talosconfig
```

## Test connectivity to the Kubernetes cluster

```sh
$ kubectl get nodes
```

## Install Cilium without proxy

The kube proxy and the cni must be disabled, we will be using cilium without kube-proxy (patch-talos-cni-and-proxy.yaml):
```yaml
cluster:
  network:
    cni:
      name: none
  proxy:
    disabled: true
```

```sh
$ talosctl --nodes 192.168.x.x --endpoints 192.168.x.x --talosconfig=./talosconfig patch machineconfig --patch @patch-talos-cni-and-proxy.yaml
$ kubectl create namespace cilium
$ kubectl label namespace cilium pod-security.kubernetes.io/enforce=privileged
$ helm repo add cilium https://helm.cilium.io/
$ helm install \
    cilium \
    cilium/cilium \
    --version 1.17.5 \
    --namespace cilium \
    --set ipam.mode=kubernetes \
    --set kubeProxyReplacement=true \
    --set securityContext.capabilities.ciliumAgent="{CHOWN,KILL,NET_ADMIN,NET_RAW,IPC_LOCK,SYS_ADMIN,SYS_RESOURCE,DAC_OVERRIDE,FOWNER,SETGID,SETUID}" \
    --set securityContext.capabilities.cleanCiliumState="{NET_ADMIN,SYS_ADMIN,SYS_RESOURCE}" \
    --set cgroup.autoMount.enabled=false \
    --set cgroup.hostRoot=/sys/fs/cgroup \
    --set k8sServiceHost=localhost \
    --set k8sServicePort=7445 \
    --set=gatewayAPI.enabled=true \
    --set=gatewayAPI.enableAlpn=true \
    --set=gatewayAPI.enableAppProtocol=true
$ kubectl patch deployment -n cilium cilium-operator --patch '{"spec": {"replicas": 1}}'
```

## Install MetalLB

```sh
$ kubectl create namespace metallb
$ kubectl label namespace metallb pod-security.kubernetes.io/enforce=privileged
$ helm repo add metallb https://metallb.github.io/metallb
$ helm install metallb metallb/metallb --namespace metallb
```

## Configure MetalLB

IP Pool et advertisement (metallb.yaml):
```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: pool
  namespace: metallb
spec:
  addresses:
  - 192.168.x.x-192.168.x.y
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: advertisement
  namespace: metallb
```

```sh
$ k apply -f metallb.yaml
```

## Eventually reduce the number of coredns pods to 1

```sh
$ kubectl patch deployment -n kube-system coredns --patch '{"spec": {"replicas": 1}}'
```

# Kubevirt

## Deployment of the local path provisioner

### Creation of the volume on the local disk

Create the machine configuration patch as `patch-talos-local-path-provisioner-uservolume.yaml`:
```yaml
apiVersion: v1alpha1
kind: UserVolumeConfig
name: local-path-provisioner
provisioning:
  diskSelector:
    match: disk.model == "TOSHIBA MQ01ABF0"
  minSize: 200GiB
  maxSize: 200GiB
```

Then apply the patch:
```sh
$ talosctl patch machineconfig --nodes 192.168.x.x --endpoints 192.168.x.x --talosconfig=./talosconfig --patch @patch-talos-local-path-provisioner-uservolume.yaml
```

### Deployment of the Kubernetes controller

Create the following manifest in the file `kustomization.yaml`:
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- github.com/rancher/local-path-provisioner/deploy?ref=v0.0.31
patches:
- patch: |-
    kind: ConfigMap
    apiVersion: v1
    metadata:
      name: local-path-config
      namespace: local-path-storage
    data:
      config.json: |-
        {
                "nodePathMap":[
                {
                        "node":"DEFAULT_PATH_FOR_NON_LISTED_NODES",
                        "paths":["/var/mnt/local-path-provisioner"]
                }
                ]
        }
- patch: |-
    apiVersion: storage.k8s.io/v1
    kind: StorageClass
    metadata:
      name: local-path
      annotations:
        storageclass.kubernetes.io/is-default-class: "true"
- patch: |-
    apiVersion: v1
    kind: Namespace
    metadata:
      name: local-path-storage
      labels:
        pod-security.kubernetes.io/enforce: privileged
```

Now we can apply this manifest with `kustomize`:

```sh
$ kustomize build | kubectl apply -f -
```

## Install the NFS CSI driver

### Install the Helm chart

We are going to use Helm to install the driver:
```sh
$ kubectl create namespace csi-driver-nfs
$ kubectl label namespace csi-driver-nfs pod-security.kubernetes.io/enforce=privileged
$ helm repo add csi-driver-nfs https://raw.githubusercontent.com/kubernetes-csi/csi-driver-nfs/master/charts
$ helm install csi-driver-nfs csi-driver-nfs/csi-driver-nfs --namespace csi-driver-nfs --version 4.11.0
```

### Configure the Storage Class

Create the manifest:
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-csi
provisioner: nfs.csi.k8s.io
parameters:
  server: 192.168.x.z
  share: /volume1/kubernetes_csi
reclaimPolicy: Delete
volumeBindingMode: Immediate
mountOptions:
  - nfsvers=4
  - nolock
allowVolumeExpansion: true
```

### Apply it

```sh
$ kubectl apply -f storageclass.yaml
```

## Install virtctl

```sh
$ export VERSION=$(curl https://storage.googleapis.com/kubevirt-prow/release/kubevirt/kubevirt/stable.txt)
$ curl -Ls https://github.com/kubevirt/kubevirt/releases/download/${VERSION}/virtctl-${VERSION}-linux-amd64 -o virtctl
$ chmod +x virtctl
$ mv virtctlt xxxx # move in $PATH

```

## Install KubeVirt

### Install the Operator

```sh
kubectl apply -f https://github.com/kubevirt/kubevirt/releases/latest/download/kubevirt-operator.yaml
```

### Install the CR to setup KubeVirt

Create the manifest kubevirt.yaml:
```yaml
apiVersion: kubevirt.io/v1
kind: KubeVirt
metadata:
  name: kubevirt
  namespace: kubevirt
spec:
  configuration:
    developerConfiguration:
      featureGates:
        - LiveMigration
        - NetworkBindingPlugins
    smbios:
      sku: "TalosCloud"
      version: "v0.1.0"
      manufacturer: "Talos Virtualization"
      product: "talosvm"
      family: "ccio"
  workloadUpdateStrategy:
    workloadUpdateMethods:
    - LiveMigrate
```

Apply the manifest:
```sh
kubectl apply -f kubevirt.yaml
```

## Install CDI

CDI is needed to import virtual disk images in our KubeVirt cluster.

### Install the Operator

```sh
kubectl apply -f https://github.com/kubevirt/kubevirt/releases/latest/download/kubevirt-operator.yaml
kubectl apply -f https://github.com/kubevirt/containerized-data-importer/releases/latest/download/cdi-operator.yaml
```

### Set a CR to configure it

Create the manifest cdi.yaml:
```yaml
apiVersion: cdi.kubevirt.io/v1beta1
kind: CDI
metadata:
  name: cdi
spec:
  config:
    scratchSpaceStorageClass: local-path
    podResourceRequirements:
      requests:
        cpu: "100m"
        memory: "60M"
      limits:
        cpu: "750m"
        memory: "2Gi"
```

Apply the manifest:
```sh
kubectl apply -f cdi.yaml
```

### Test a VM

Apply the manifest (change the SSH public key with yours):
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: vm
---
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: fedora-vm
  namespace: vm
spec:
  running: false
  template:
    metadata:
      labels:
        kubevirt.io/vm: fedora-vm
      annotations:
        kubevirt.io/allow-pod-bridge-network-live-migration: "true"

    spec:
      evictionStrategy: LiveMigrate
      domain:
        cpu:
          cores: 2
        resources:
          requests:
            memory: 4G
        devices:
          disks:
            - name: fedora-vm-pvc
              disk:
                bus: virtio
            - name: cloudinitdisk
              disk:
                bus: virtio
          interfaces:
          - name: podnet
            masquerade: {}
      networks:
        - name: podnet
          pod: {}
      volumes:
        - name: fedora-vm-pvc
          persistentVolumeClaim:
            claimName: fedora-vm-pvc
        - name: cloudinitdisk
          cloudInitNoCloud:
            networkData: |
              network:
                version: 1
                config:
                  - type: physical
                    name: eth0
                    subnets:
                      - type: dhcp
            userData: |-
              #cloud-config
              users:
                - name: cloud-user
                  ssh_authorized_keys:
                    - ssh-rsa ....
                  sudo: ['ALL=(ALL) NOPASSWD:ALL']
                  groups: sudo
                  shell: /bin/bash
              runcmd:
                - "sudo touch /root/installed"
                - "sudo dnf update"
                - "sudo dnf install httpd fastfetch -y"
                - "sudo systemctl daemon-reload"
                - "sudo systemctl enable httpd"
                - "sudo systemctl start --no-block httpd"

  dataVolumeTemplates:
  - metadata:
      name: fedora-vm-pvc
    spec:
      storage:
        resources:
          requests:
            storage: 35Gi
        accessModes:
          - ReadWriteMany
        storageClassName: "nfs-csi"
      source:
        http:
          url: "https://fedora.mirror.wearetriple.com/linux/releases/40/Cloud/x86_64/images/Fedora-Cloud-Base-Generic.x86_64-40-1.14.qcow2"
---
apiVersion: v1
kind: Service
metadata:
  labels:
    kubevirt.io/vm: fedora-vm
  name: fedora-vm
  namespace: vm
spec:
  ipFamilyPolicy: PreferDualStack
  externalTrafficPolicy: Local
  ports:
  - name: ssh
    port: 22
    protocol: TCP
    targetPort: 22
  - name: httpd
    port: 80
    protocol: TCP
    targetPort: 80
  selector:
    kubevirt.io/vm: fedora-vm
  type: LoadBalancer
```

Then start the VM:
```sh
$ virtctl start fedora-vm -n vm
```

Check that the VM is started:
```sh
$ kubectl describe pod -n vm -l kubevirt.io/vm=fedora-vm
```

### Upload a qcow2

Expose the cdi-uploadproxy pod with a nodeport:
```yaml
apiVersion: v1
kind: Service
metadata:
 name: cdi-uploadproxy-nodeport
 namespace: cdi
 labels:
   cdi.kubevirt.io: "cdi-uploadproxy"
spec:
 type: NodePort
 ports:
   - port: 443
     targetPort: 8443
     nodePort: 30085
     protocol: TCP
 selector:
   cdi.kubevirt.io: cdi-uploadproxy
```

Then upload the qcow2 to the pvc with `virtctl image-upload`:
```sh
$ virtctl image-upload dv win10 --force-bind --size=80Gi --volume-mode=filesystem --access-mode=ReadWriteOncePod --image-path=${HOME}/win10.qcow2 --uploadproxy-url=https://192.168.x.x:30085 --insecure -n vm
```

Then launch the VM:
```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: win10
  namespace: vm
spec:
  running: false
  template:
    metadata:
      labels:
        kubevirt.io/vm: win10
      annotations:
        kubevirt.io/allow-pod-bridge-network-live-migration: "true"
    spec:
      evictionStrategy: LiveMigrate
      domain:
        cpu:
          cores: 2
        resources:
          requests:
            memory: 4G
        devices:
          disks:
            - name: win10
              disk:
                bus: virtio
          interfaces:
          - name: podnet
            masquerade: {}
      networks:
        - name: podnet
          pod: {}
      volumes:
        - dataVolume:
            name: win10
          name: win10
---
apiVersion: v1
kind: Service
metadata:
  labels:
    kubevirt.io/vm: win10
  name: win10
  namespace: vm
spec:
  ipFamilyPolicy: PreferDualStack
  externalTrafficPolicy: Local
  ports:
  - name: rdp
    port: 3389
    protocol: TCP
    targetPort: 3389
  selector:
    kubevirt.io/vm: win10
  type: LoadBalancer
```

to expose the RDP port, we can also use the command:
```sh
$ virtctl expose vmi -n vm win10 --port=3389 --name=win10 --type=LoadBalancer
```

# Talos debug

## Launch a DaemonSet on each node to get a real linux, and be able to troubleshoot:

### Launch the debug DaemonSet

```sh
$ kubectl apply -f debug/talos-debug-tools.yml
```

### Connect to it

```sh
$ kubectl exec -n kube-system -it ds/sshd -- bash
```

It should be available by ssh also, but we need to expose SSH for that or use port-forward.

### Destroy it

```sh
$ kubectl delete -f talos-debug-tools.yml
```

## Recreate a talosconfig if the certificate is not valid anymore

```sh
$ talosctl gen config --with-secrets secrets.yaml --output-types talosconfig -o talosconfig --force myKluster https://192.168.x.x:6443
```
