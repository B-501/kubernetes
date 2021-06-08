#### Kubernetes CSI Ceph Cephfs

##### in ceph

- 1

  ```bash
  ceph mon dump
  ```

- 2

  ```bash
  ceph fs volume create kubernetes-cephfs
  ```

- 3

  ```bash
  ceph auth get-or-create client.cephfs mon 'allow r' osd 'allow rwx pool=kubernetes-cephfs'
  ```

- 4

  ```bash
  ceph mds stat
  ```

##### in kubernetes

- 1

  ```bash
  vi csi-cephfs-config-map.yaml
  ```

  ```bash
  ---
  apiVersion: v1
  kind: ConfigMap
  data:
    config.json: |-
      [
        {
          "clusterID": "fe155e37-67d6-4159-9327-0bd8cabe8022",
          "monitors": [
            "172.16.18.211:6789",
            "172.16.18.212:6789",
            "172.16.18.213:6789"
          ],
          "cephFS": {
            "subvolumeGroup": "kubernetes-cephfs"
          }
        }
      ]
  metadata:
    name: csi-cephfs-config-map
  ```

- 2

  ```bash
  kubectl apply -f csi-cephfs-config-map.yaml
  ```

- 3

  ```bash
  ceph auth get client.admin
  ceph auth get client.cephfs
  ```

- 4

  ```bash
  vi csi-cephfs-secret.yaml
  ```

  ```bash
  ---
  apiVersion: v1
  kind: Secret
  metadata:
    name: csi-cephfs-secret
    namespace: default
  stringData:
    userID: cephfs
    userKey: AQABVb9gjwl0AxAAy8VUDwwzOYLfyKeWZo2qOA==
    adminID: admin
    adminKey: AQAxz7lgpsGODRAA0Z9tMU1IvYjI1WXStbl4mg==
  ```

- 5

  ```bash
  kubectl apply -f csi-cephfs-secret.yaml
  ```

- 6

  ```bash
  vi csi-cephfs-sc.yaml
  ```

  ```bash
  ---
  apiVersion: storage.k8s.io/v1
  kind: StorageClass
  metadata:
     name: csi-cephfs-sc
  allowVolumeExpansion: true
  provisioner: cephfs.csi.ceph.com
  parameters:
     clusterID: fe155e37-67d6-4159-9327-0bd8cabe8022
     fsName: kubernetes-cephfs
     csi.storage.k8s.io/controller-expand-secret-name: csi-cephfs-secret
     csi.storage.k8s.io/controller-expand-secret-namespace: default
     csi.storage.k8s.io/provisioner-secret-name: csi-cephfs-secret
     csi.storage.k8s.io/provisioner-secret-namespace: default
     csi.storage.k8s.io/node-stage-secret-name: csi-cephfs-secret
     csi.storage.k8s.io/node-stage-secret-namespace: default
  reclaimPolicy: Delete
  mountOptions:
     - debug
  ```

- 7

  ```bash
  kubectl apply -f csi-cephfs-sc.yaml 
  ```

- 8

  ```bash
  kubectl apply -f https://raw.githubusercontent.com/ceph/ceph-csi/master/deploy/cephfs/kubernetes/csi-provisioner-rbac.yaml
  
  kubectl apply -f https://raw.githubusercontent.com/ceph/ceph-csi/master/deploy/cephfs/kubernetes/csi-nodeplugin-rbac.yaml
  
  kubectl apply -f https://raw.githubusercontent.com/ceph/ceph-csi/master/deploy/cephfs/kubernetes/csi-cephfsplugin-provisioner.yaml
  
  kubectl apply -f https://raw.githubusercontent.com/ceph/ceph-csi/master/deploy/cephfs/kubernetes/csi-cephfsplugin.yaml
  ```

- 9

  ```bash
  kubectl get pod
  ```

- 10

  ```bash
  vi csi-cephfs-pvc.yaml
  ```

  ```bash
  ---
  apiVersion: v1
  kind: PersistentVolumeClaim
  metadata:
    name: csi-cephfs-pvc
  spec:
    accessModes:
      - ReadWriteMany
    resources:
      requests:
        storage: 1Gi
    storageClassName: csi-cephfs-sc
  ```

- 11

  ```bash
  kubectl apply -f csi-cephfs-pvc.yaml
  ```

- 12

  ```bash
  kubectl get pvc
  ```
