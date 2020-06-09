## Azure File CSI Driver VHD disk feature
Attach Azure disks in < 1 second. Attach as many as you want. VHD disk(based on azure file) feature could mount Azure disks as Linux **block device** directly on VMs without dependency on the host.

### Feature Status: Beta

 - Motivation:

There are slow disk attach/detach issues on Azure managed disk(sometimes parallel disk attach/detach costs more than one minute), this feature aims to solve such slow disk attach/detach issues. With this feature, VHD disk file is created on Azure File, VHD disk file is mounted over SMB from agent node, and then vhd file is mounted as a loop block device. It could offer performance similar to a local direct-attached storage, while attach/detach disk would only costs < 1 second.

 - Advantages over managed disk
   - Attach/detach disk < 1s
   - No data disk count limit on one agent node (e.g. while max data disk count is 8 on DS2_v2 VM for managed disk)
 - Disadvantages/limitations over managed disk
   - Cost (Premium File storage pricing is one time more expensive than Premium Disk)
   - Not supported on Windows yet
   - VHD disk is unmanaged disk stored in azure storage account, since there is [IOPS limit(20K) per storage account](https://docs.microsoft.com/en-us/azure/storage/common/scalability-targets-standard-account#scale-targets-for-standard-storage-accounts), user needs to control total quota according to VHD disk num.

 - Performance test have done

Scheduling 20 pods with one vhd disk each on **one** node **in parallel** could be completed in 30s, while for azure managed disk driver, it's 30min. (not including the disk create/format and image pulling time)

 - How to use
 
Add a new parameter(`fsType`) in Azure File CSI driver storage class, other parameters are same as [`file.csi.azure.com` driver parameters](../../../docs/driver-parameters.md)

```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: azurefile-csi
provisioner: file.csi.azure.com
parameters:
  skuName: Premium_LRS  # available values: Standard_LRS, Standard_GRS, Standard_ZRS, Standard_RAGRS, Premium_LRS
  fsType: ext4  # available values: ext4, ext3, ext2, xfs
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-azurefile
spec:
  accessModes:
    - ReadWriteOnce  # available values: ReadWriteOnce, ReadOnlyMany
  resources:
    requests:
      storage: 100Gi
  storageClassName: azurefile-csi
```

#### Prerequisite
 - [install azurefile csi driver](https://github.com/kubernetes-sigs/azurefile-csi-driver/blob/master/docs/install-azurefile-csi-driver.md)

#### Example#1. create a pod with vhd disk mount on Linux
##### Option#1: Dynamic Provisioning
 - Create an azurefile CSI storage class and PVC
```console
kubectl create -f https://raw.githubusercontent.com/kubernetes-sigs/azurefile-csi-driver/master/deploy/example/disk/storageclass-azurefile-csi.yaml
kubectl create -f https://raw.githubusercontent.com/kubernetes-sigs/azurefile-csi-driver/master/deploy/example/disk/pvc-azurefile-disk.yaml
```

##### Option#2: Static Provisioning(use an existing vhd file in azure file share)
> make sure credential in cluster could access that file share
 - Create an azurefile CSI storage class and PVC
```console
kubectl create -f https://raw.githubusercontent.com/kubernetes-sigs/azurefile-csi-driver/master/deploy/example/disk/storageclass-azurefile-existing-disk.yaml
kubectl create -f https://raw.githubusercontent.com/kubernetes-sigs/azurefile-csi-driver/master/deploy/example/disk/pvc-azurefile-disk.yaml
```

#### 2. validate PVC status and create an nginx pod
 - make sure pvc is created and in `Bound` status finally
```console
watch kubectl describe pvc pvc-azurefile
```

 - create a pod with azurefile CSI PVC
```console
kubectl create -f https://raw.githubusercontent.com/kubernetes-sigs/azurefile-csi-driver/master/deploy/example/nginx-pod-azurefile.yaml
```

#### 3. enter the pod container to do validation
 - watch the status of pod until its Status changed from `Pending` to `Running` and then enter the pod container
```console
$ kubect exec -it nginx-azurefile bash
# df -h
Filesystem      Size  Used Avail Use% Mounted on
...
/dev/loop0       98G   61M   98G   1% /mnt/azurefile
/dev/sda1        29G   16G   14G  53% /etc/hosts
...
```
In the above example, there is a `/mnt/azurefile` directory mounted as ext4 filesystem.

#### Example#2. create 5 pods with vhd disk mount in parallel
```console
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/azurefile-csi-driver/master/deploy/example/disk/statefulset-stress.yaml
```
 - scale pod replicas
```console
kubectl scale statefulset statefulset-azurefile --replicas=30
```
> note: create multiple vhd disks in parallel in one storage account may cause IO throttling, user could set `storageAccount` to specify different storage accounts for different vhd disks

```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: azurefile-csi
provisioner: file.csi.azure.com
parameters:
  storageAccount: EXISTING_STORAGE_ACCOUNT_NAME
  fsType: ext4  # available values: ext4, ext3, ext2, xfs
```
