This document outlines the instructions to have a process in place for Disaster Recovery(DR) using AWS and Kubernetes.
It involves two steps.

1. First have a plan to take regular backups using **AWS Lifecycle Manager**
    * use proper tags for which you want to take snapshots
    * Provide the Schedule details like how often you want the snapshots to be taken
    * Retention policy - how many hours/days you want to keep the snapshots
2. Follow below steps to perform recovery from the snapshots.
    * Scale down replicas to 0 for the statefulset you want to recover from snapshot
       * ``` kubectl scale statefulset <statefulset-name> --replicas=0 -n <namespace> ```
    * Delete existing PVCs for which we want to restore the snapshot.
      * ``` kubectl delete pvc <pvc-name> -n <namespace> ```
    **Note:** If this command get stuck, it could be because of pvc-protection policy on PVC, need to delete the "finalizers" section from PVC configuration.
    * Create a volume from EBS snapshot by providing size, tags and AZ requirements.
    * EC2 -> Snapshots -> Select the snapshot you want to create a volume from -> Click on "Create volume from snapshot" from the Actions menu.
    * Create a PersistentVolume(PV) in Kubernetes by referring the EBS Volume you created on above step. Provide NodeAffinity and Zone configuration if required
      * Create a yaml file with below content
      ```
      apiVersion: v1
      kind: PersistentVolume
      metadata:
        name: restored-pv-kafka-broker-0
      spec:
        capacity:
          storage: 20Gi
        csi:
          driver: <driver>
          volumeHandle: <EBS-Volume-Id>
          fsType: ext4
        accessModes:
          - ReadWriteOnce
        persistentVolumeReclaimPolicy: Retain
        storageClassName: <StorageClass>
        volumeMode: Filesystem
        nodeAffinity:
          required:
            nodeSelectorTerms:
              - matchExpressions:
                  - key: <topology.ebs.csi.aws.com/zone>
                    operator: In
                    values:
                      - <zone-id>

      ```
      * Create PV by running below kubectl command
        ``` kubectl apply -f <filename>.yaml ```
    * We need to create a PersistentVolumeClaim by referring to the PV we created on above step
      * Create a yaml file with below content
        ```
        apiVersion: v1
        kind: PersistentVolumeClaim
        metadata:
          name: restored-kafka-broker-0
          namespace: <namespace>
          labels:
            app.kubernetes.io/component: broker
            app.kubernetes.io/name: kafka
            app.kubernetes.io/part-of: kafka
        spec:
          accessModes:
            - ReadWriteOnce
          resources:
            requests:
              storage: 20Gi // depends on PV
          storageClassName: <StorageClass>
          volumeName: restored-pv-kafka-broker-0 // PV name from previous step

        ```
      * Create PV by running below kubectl command
        ``` kubectl apply -f <filename>.yaml ```
    * Scale up the replicas to required replicas for the statefulset

