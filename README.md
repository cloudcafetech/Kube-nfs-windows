# Kubernetes nfs Volume Driver for windows
A simple volume driver based on [Kubernetes' Flexvolume](https://github.com/kubernetes/community/blob/master/contributors/devel/flexvolume.md) that allows Kubernetes hosts to mount nfs volumes (nfs shares) into pods and containers.

It has been tested under Kubernetes versions:

* AKS 1.18.x

## DaemonSet Installation
When dealing with a large cluster, manually copying the driver to all hosts becomes inhuman. As proposed in Flexvolume’s documentation, the recommended driver deployment method is to have a DaemonSet install the driver cluster-wide automatically.

A Docker image `mcr.microsoft.com/powershell:7.1.0-preview.5-nanoserver-1809` is available for this purpose, which can be deployed into a Kubernetes cluster using the `winnfs-flex-volume.yaml` from this repository. The image is built `FROM nanoserver`, so the it’s essentially very small.

```
https://github.com/cloudcafetech/kube-nfs-windows.git
cd kube-nfs-windows
kubectl create -f winnfs-flex-volume.yaml
```

> NOTE: Note: This deployment automatically installs host dependencies and does not need to be completed manually on all hosts.

If you need to tweak or customize the installation, you can modify the `winnfs-flex-volume.yaml` directly.

Installing is a one time job. So, once you have verified that it’s completed, the DaemonSet can be safely removed.

```kubectl delete -f winnfs-flex-volume.yaml```

## The Volume Plugin Directory
As of today with Kubernetes v1.16, the kubelet’s default directory for volume plugins is `/usr/libexec/kubernetes/kubelet-plugins/volume/exec/` (for Linux node),  `C:\usr\libexec\kubernetes\kubelet-plugins\volume\exec` (for Windows node)in general this folder does not exit by deploying `winnfs-flex-volume.yaml` daemonset will create folder.  This could be different if your installation changed this directory using the `--volume-plugin-dir` parameter.

Please, review the kubelet command line parameters (namely `--volume-plugin-dir`) and make sure it matches the directory where the driver will be installed.

You can modify `winnfs-flex-volume.yaml` and change the field `spec.template.spec.volumes.hostPath.path` to the path used by your Kubernetes installation.

## Customizing the Vendor/Driver name
By default, the driver installation path is `$KUBELET_PLUGIN_DIRECTORY/nfs-win~nfs.cmd`.

For some installations, you may need to change the vendor + driver name. You can define the installed vendor name/directory by adjusting the `winnfs-flex-volume.yaml` plugin directory.

```yaml

## snippet ##

      containers:
        - image: mcr.microsoft.com/powershell:7.1.0-preview.5-nanoserver-1809
          env:
          - name: DRIVER
            value: nfs-win
          command: 
          - pwsh.exe
          args:
          - /Command
          - mkdir -Force c:\host\$env:DRIVER~nfs.cmd;
          - cp -Force c:\kubelet-plugins\flexvolume.ps1 c:\host\$env:DRIVER~nfs.cmd;
          - cp -Force c:\kubelet-plugins\nfs.cmd c:\host\$env:DRIVER~nfs.cmd;
          - cp -Force c:\kubelet-plugins\nfs.ps1 c:\host\$env:DRIVER~nfs.cmd;
          - cat c:\kubelet-plugins\Readme.md;
          - while ($true) {start-sleep -s 3600}

## snippet ##

```

The example above will install the driver in the path `$KUBELET_PLUGIN_DIRECTORY/nfs-win~nfs.cmd/nfs.cmd`. For the most part, changig the `DRIVER` variable should be enough to make your installation unique to your needs.

## Example of Deployment
The following is an example of Deployment that uses the volume driver.

Edit below with NFS server and share details and deploy. And make sure sufficient permission in NFS share in NFS server (for UNIX it should be 777).

```
PVCNAME=winfs
cat <<EOF > nfs-flex-vol-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
 name: $PVCNAME
spec:
 storageClassName: kubenfs-storage
 accessModes:
  - ReadWriteMany
 resources:
   requests:
     storage: 50Mi
EOF
kubectl create -f nfs-flex-vol-pvc.yaml
sleep 25
NS=$(kubectl describe pvc $PVCNAME | grep -e Namespace: | cut -f2 -d : | sed 's/ //g')
NV=$(kubectl describe pvc $PVCNAME | grep -e Name: -e Volume: | cut -f2 -d : | sed 's/ //g' | sed ':a; N; $!ba; s/\n/-/g')
VOLNAME=$NS-$NV
cat <<EOF > nfs-flex-vol-demo.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nfs-flex-vol
  labels:
    app: nfs-flex-vol
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nfs-flex-vol
  template:
    metadata:
      labels:
        app: nfs-flex-vol
    spec:
      nodeSelector:
        beta.kubernetes.io/os: windows
      tolerations:
      - key: "os"
        operator: "Equal"
        value: "windows"
        effect: "NoSchedule"
      containers:
      - name: nfs-flex-vol
        image: mcr.microsoft.com/powershell:7.1.0-preview.5-nanoserver-1809
        imagePullPolicy: IfNotPresent
        command:
        - pwsh.exe
        args:
        - /Command
        - Write-Output "\$env:POD_NAME Started on \$env:NODENAME at \$(Get-Date)" >> /d/test.txt;
        - ping -t 127.0.0.1 >> /d/test.txt
        env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: NODENAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
        volumeMounts:
        - name: nfs-volume
          mountPath: /d
      volumes:
      - name: nfs-volume
        flexVolume:
          driver: "nfs-win/nfs.cmd"
          options:
            # source should be following formats
            # nfs://servername/share/path
            source: "nfs://10.20.1.4/var/nfs/general/$VOLNAME/"
EOF
```

### Reference

https://www.yfdou.com/archives/kubernetes_nfs_Volume-_driver_for_windows.html


