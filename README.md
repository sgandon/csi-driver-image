# csi-driver-image

This is a CSI driver for mounting images as PVs or ephemeral volumes.

Our first aim is resource saving. No redundant image pull and store. No new containers. 
The driver shares image store with container runtime. 
It uses CRI to pull images, and mount images via snapshot service of runtime. 
All requests to mount the same image also share the same snapshot.

What we can do further may be building images via the snapshot capability of CSI.
It assures that new created images also exist in the current runtime and can be run immediately,
especially those images under developing, just like what Docker does.
If Docker is replaced by some others, this driver can offer the same experience to Docker. 

Currently, a possible alternate is **buildkit**. It already has containerd as a backend worker.   

## Install

### Containerd

Containerd is our recommend container runtime. It is the most compatible runtime to the driver. 

```shell script
kubectl apply -f https://raw.githubusercontent.com/warm-metal/csi-driver-image/master/install/cri-containerd.yaml
```

### Docker

As mentioned before, CRI is required to pull images.
You can enable it by adding `--cri-containerd` for docker daemon or `--docker-opt="cri-containerd=true"` for minikube.

Until Docker supports [image sharing](https://github.com/moby/moby/issues/38043) with its builtin containerd, We don't encourage this sort of usage.
It means that the driver can't use images managed by Docker daemon. 

```shell script
kubectl apply -f https://raw.githubusercontent.com/warm-metal/csi-driver-image/master/install/cri-docker.yaml
```

### Cluster with custom configuration

Some K8s clusters installed with custom configurations, such as cluster installed via microk8s.
For these clusters, provided manifests for both docker andcri are also available after modifying some hostpaths.

In the `volumes` section of the manifest, 
1. Replace `/var/lib/kubelet` with `root-dir` of kubelet,
2. Replace `/run/containerd.sock` with your containerd socket path.

```yaml
      ...
      volumes:
        - hostPath:
            path: /var/lib/kubelet/plugins/csi-image.warm-metal.tech
            type: DirectoryOrCreate
          name: socket-dir
        - hostPath:
            path: /var/lib/kubelet/pods
            type: DirectoryOrCreate
          name: mountpoint-dir
        - hostPath:
            path: /var/lib/kubelet/plugins_registry
            type: Directory
          name: registration-dir
        - hostPath:
            path: /
            type: Directory
            name: host-rootfs
        - hostPath:
            path: /run/containerd/containerd.sock
            type: Socket
          name: runtime-socket
```

## Usage

Provided manifests will install a CSIDriver `csi-image.warm-metal.tech` and a DaemonSet.
You can mount images as either pre-provisioned PVs or ephemeral volumes.

As a ephemeral volume, `volumeAttributes` are **image**(required), **secret**, and **pullAlways**.

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: ephemeral-volume
spec:
  template:
    metadata:
      name: ephemeral-volume
    spec:
      containers:
        - name: ephemeral-volume
          image: docker.io/warmmetal/csi-image-test:check-fs
          env:
            - name: TARGET
              value: /target
          volumeMounts:
            - mountPath: /target
              name: target
      restartPolicy: Never
      volumes:
        - name: target
          csi:
            driver: csi-image.warm-metal.tech
            volumeAttributes:
              image: "docker.io/warmmetal/csi-image-test:simple-fs"
              # set pullAlways if you want to ignore local images
              # pullAlways: "true"
              # set secret if the image is private
              # secret: "pull image secret name"
  backoffLimit: 0
```

For a PV, the `volumeHandle` instead the attribute **image**, specify the target image.

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-test-csi-image-test-simple-fs
spec:
  storageClassName: csi-image.warm-metal.tech
  capacity:
    storage: 5Gi
  accessModes:
    - ReadOnlyMany
  persistentVolumeReclaimPolicy: Retain
  csi:
    driver: csi-image.warm-metal.tech
    volumeHandle: "docker.io/warmmetal/csi-image-test:simple-fs"
    # volumeAttributes:
      # set pullAlways if you want to ignore local images
      # pullAlways: "true"
      # set secret if the image is private
      # secret: "pull image secret name"
```

See all [examples](https://github.com/warm-metal/csi-driver-image/tree/master/test/integration/manifests).
