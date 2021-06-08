#### Kubernetes CSI Ceph RBD

##### in ceph

- 1

  ```bash
  ceph osd pool create kubernetes
  ```

- 2

  ```bash
  rbd pool init kubernetes
  ```

- 3

  ```bash
  ceph auth get-or-create client.kubernetes mon 'profile rbd' osd 'profile rbd pool=kubernetes' mgr 'profile rbd pool=kubernetes'
  ```

- 4

  ```bash
  ceph mon dump
  ```

##### in kubernetes

- 5

  ```bash
  vi csi-config-map.yaml
  ```

  ```yaml
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
          ]
        }
      ]
  metadata:
    name: ceph-csi-config
  ```

- 6

  ```bash
  kubectl apply -f csi-config-map.yaml
  ```

- 7

  ```bash
  vi csi-kms-config-map.yaml
  ```

  ```bash
  ---
  apiVersion: v1
  kind: ConfigMap
  data:
    config.json: |-
      {}
  metadata:
    name: ceph-csi-encryption-kms-config
  ```

- 8

  ```bash
  kubectl apply -f csi-kms-config-map.yaml
  ```

- 9

  ```bash
  vi csi-rbd-secret.yaml
  ```

  ```bash
  ---
  apiVersion: v1
  kind: Secret
  metadata:
    name: csi-rbd-secret
    namespace: default
  stringData:
    userID: kubernetes
    userKey: AQCFw71gY2WfHxAAE5DLDbm8c93z3DwAHCZLaQ==
  ```

- 10

  ```bash
  kubectl apply -f csi-rbd-secret.yaml
  ```

- 11

  ```bash
  wget https://raw.githubusercontent.com/ceph/ceph-csi/master/deploy/rbd/kubernetes/csi-provisioner-rbac.yaml
  kubectl apply -f csi-provisioner-rbac.yaml
  wget https://raw.githubusercontent.com/ceph/ceph-csi/master/deploy/rbd/kubernetes/csi-nodeplugin-rbac.yaml
  kubectl apply -f csi-nodeplugin-rbac.yaml
  ```

- 12

  ```bash
  wget https://raw.githubusercontent.com/ceph/ceph-csi/master/deploy/rbd/kubernetes/csi-rbdplugin-provisioner.yaml
  kubectl apply -f csi-rbdplugin-provisioner.yaml
  wget https://raw.githubusercontent.com/ceph/ceph-csi/master/deploy/rbd/kubernetes/csi-rbdplugin.yaml
  kubectl apply -f csi-rbdplugin.yaml
  ```

- 13

  ```bash
  vi csi-rbd-sc.yaml
  ```

  ```bash
  ---
  apiVersion: storage.k8s.io/v1
  kind: StorageClass
  metadata:
     name: csi-rbd-sc
  provisioner: rbd.csi.ceph.com
  parameters:
     clusterID: fe155e37-67d6-4159-9327-0bd8cabe8022
     pool: kubernetes
     imageFeatures: layering
     csi.storage.k8s.io/provisioner-secret-name: csi-rbd-secret
     csi.storage.k8s.io/provisioner-secret-namespace: default
     csi.storage.k8s.io/controller-expand-secret-name: csi-rbd-secret
     csi.storage.k8s.io/controller-expand-secret-namespace: default
     csi.storage.k8s.io/node-stage-secret-name: csi-rbd-secret
     csi.storage.k8s.io/node-stage-secret-namespace: default
  reclaimPolicy: Delete
  allowVolumeExpansion: true
  mountOptions:
     - discard
  ```

- 14

  ```bash
  kubectl apply -f csi-rbd-sc.yaml
  ```

- 15

  ```bash
  vi raw-block-pvc.yaml
  ```

  ```bash
  ---
  apiVersion: v1
  kind: PersistentVolumeClaim
  metadata:
    name: raw-block-pvc
  spec:
    accessModes:
      - ReadWriteOnce
    volumeMode: Block
    resources:
      requests:
        storage: 1Gi
    storageClassName: csi-rbd-sc
  ```

- 16

  ```bash
  kubectl apply -f raw-block-pvc.yaml
  ```

- 17

  ```bash
  vi raw-block-pod.yaml
  ```

  ```bash
  ---
  apiVersion: v1
  kind: Pod
  metadata:
    name: pod-with-raw-block-volume
  spec:
    containers:
      - name: fc-container
        image: fedora:26
        command: ["/bin/sh", "-c"]
        args: ["tail -f /dev/null"]
        volumeDevices:
          - name: data
            devicePath: /dev/xvda
    volumes:
      - name: data
        persistentVolumeClaim:
          claimName: raw-block-pvc
  ```

- 18

  ```bash
  kubectl apply -f raw-block-pod.yaml
  ```

- 19

  ```bash
  vi pvc.yaml
  ```

  ```bash
  ---
  apiVersion: v1
  kind: PersistentVolumeClaim
  metadata:
    name: rbd-pvc
  spec:
    accessModes:
      - ReadWriteOnce
    volumeMode: Filesystem
    resources:
      requests:
        storage: 1Gi
    storageClassName: csi-rbd-sc
  ```

- 20

  ```bash
  kubectl apply -f pvc.yaml
  ```

- 21

  ```bash
  vi pod.yaml
  ```

  ```bash
  ---
  apiVersion: v1
  kind: Pod
  metadata:
    name: csi-rbd-demo-pod
  spec:
    containers:
      - name: web-server
        image: nginx
        volumeMounts:
          - name: mypvc
            mountPath: /var/lib/www/html
    volumes:
      - name: mypvc
        persistentVolumeClaim:
          claimName: rbd-pvc
          readOnly: false
  ```

- 22

  ```bash
  kubectl apply -f pod.yaml
  ```
