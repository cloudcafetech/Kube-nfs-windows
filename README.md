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

Edit `demo-nfs-flex-volume.yaml` with NFS server and share details and deploy. And make sure sufficient permission in NFS share in NFS server (for UNIX it should be 777).

```kubectl create -f demo-nfs-flex-volume.yaml```

If Kubernetes PV & PVC style then use following deployment.

```kubectl create -f win-nfs-flexvol-demo.yaml```

### Reference

https://www.yfdou.com/archives/kubernetes_nfs_Volume-_driver_for_windows.html


