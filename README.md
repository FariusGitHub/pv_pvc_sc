# Kubernetes Storage

![](/images/11-image01.png)

In the context of Kubernetes storage, PV (Persistent Volume), PVC (Persistent Volume Claim), Storage Class, and Pod are all interconnected components.

1. Persistent Volume (PV): PV is a cluster-wide resource that represents a piece of storage in the cluster. It can be a physical disk, a network-attached storage, or any other storage resource. PVs are provisioned by the cluster administrator and made available to the users.

2. Persistent Volume Claim (PVC): PVC is a request made by a user to use a specific amount of storage from the available PVs. It acts as a request for a specific type of storage resource with specific capacity and access modes. PVCs are created by users and are bound to a PV that matches the requested criteria.

3. Storage Class: Storage Class is an abstraction layer that defines the types of storage available in the cluster. It provides a way for users to dynamically provision PVs without having to manually create them. Storage Classes define the parameters for provisioning PVs, such as the provisioner, reclaim policy, and other parameters.

4. Pod: A Pod is the smallest and most basic unit in Kubernetes. It represents a single instance of a running process in the cluster. Pods can have one or more containers running inside them. In the context of storage, a Pod can request storage resources by creating a PVC and using it as a volume in the Pod's container.

The connections between these components are as follows:

- A user creates a PVC to request a specific amount and type of storage.
- The PVC is bound to a PV that matches the requested criteria. The binding is done based on the Storage Class specified in the PVC.
- The PV is provisioned by the cluster administrator or dynamically provisioned by the Storage Class.
- The Pod can then use the PVC as a volume to access the requested storage. The Pod's container can read from and write to this volume as needed.

PV, PVC, Storage Class, and Pod work together to provide a flexible and scalable storage solution in Kubernetes, allowing users to request and use storage resources in a controlled and efficient manner.

## Storage Class

Let's create some new storage class and tweak some parameter inside

```txt
$ cat storage-class.yaml
  apiVersion: storage.k8s.io/v1
  kind: StorageClass
  metadata:
    name: my-storage-class
  provisioner: kubernetes.io/no-provisioner
  volumeBindingMode: Immediate
  allowVolumeExpansion: True

$ kubectl get sc
NAME                   PROVISIONER                    RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
my-storage-class       kubernetes.io/no-provisioner   Delete          Immediate              true                  8s
```

## PVC
Similarly we can create a simple PVC as follow. Initially the pvc status of pvc-azure below would be pending. As soon as a new deployment assigning claimName: pvc-azure inside the volume, the pvc status turned into Bound.

```txt
$ cat pvc.yaml
  apiVersion: v1
  kind: PersistentVolumeClaim
  metadata:
    name: pvc-azure
  spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: local-path
  resources:
    requests:
      storage: 1Gi

$ kubectl apply -f pvc.yaml
  persistentvolumeclaim/pvc-azure created

$ kubectl get pvc
  NAME        STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
  pvc-azure   Pending                                      local-path     5s

$ cat pod.yaml
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: nginx-azure-deployment
  spec:
    replicas: 1
    selector:
      matchLabels:
        app: nginx
    template:
      metadata:
        labels:
          app: nginx
      spec:
        volumes:
        - name: myservercontent
          persistentVolumeClaim:
            claimName: pvc-azure
        containers:
        - name: nginx
          image: nginx
          ports:
          - containerPort: 80
          volumeMounts:
          - name: myservercontent
            mountPath: "/usr/share/nginx/html/web-app"

$kubectl apply -f pod.yaml
  deployment.apps/nginx-azure-deployment created

$ kubectl get pvc
  NAME        STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
  pvc-azure   Bound    pvc-2d558cbc-eace-4ed6-9322-f1c345871a25   1Gi        RWO            local-path     25s

```

take a deeper look to the pod

```txt
$ kubectl get pods
  NAME                                     READY   STATUS    RESTARTS   AGE
  nginx-azure-deployment-8659d8989-f8ct2   1/1     Running   0          6m42s

$ kubectl describe pod nginx-azure-deployment-8659d8989-f8ct2
  Name:             nginx-azure-deployment-8659d8989-f8ct2
  Namespace:        default
  Priority:         0
  Service Account:  default
  Node:             master/10.73.136.111
  Start Time:       Fri, 16 Feb 2024 18:22:16 -0500
  Labels:           app=nginx
                    pod-template-hash=8659d8989
  Annotations:      <none>
  Status:           Running
  IP:               10.42.0.18
  IPs:
    IP:           10.42.0.18
  Controlled By:  ReplicaSet/nginx-azure-deployment-8659d8989
  Containers:
    nginx:
      Container ID:   containerd://115ec3383e040b28e103512a27cc20d04b8ad7e5f3701f20d1695a4f2758c8bb
      Image:          nginx
      Image ID:       docker.io/library/nginx@sha256:c26ae7472d624ba1fafd296e73cecc4f93f853088e6a9c13c0d52f6ca5865107
      Port:           80/TCP
      Host Port:      0/TCP
      State:          Running
        Started:      Fri, 16 Feb 2024 18:22:19 -0500
      Ready:          True
      Restart Count:  0
      Environment:    <none>
      Mounts:
        /usr/share/nginx/html/web-app from myservercontent (rw)
        /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-qrpv5 (ro)
  Conditions:
    Type              Status
    Initialized       True 
    Ready             True 
    ContainersReady   True 
    PodScheduled      True 
  Volumes:
    myservercontent:
      Type:       PersistentVolumeClaim (a reference to a PersistentVolumeClaim in the same namespace)
      ClaimName:  pvc-azure
      ReadOnly:   false
    kube-api-access-qrpv5:
      Type:                    Projected (a volume that contains injected data from multiple sources)
      TokenExpirationSeconds:  3607
      ConfigMapName:           kube-root-ca.crt
      ConfigMapOptional:       <nil>
      DownwardAPI:             true
  QoS Class:                   BestEffort
  Node-Selectors:              <none>
  Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                              node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
  Events:
    Type    Reason     Age    From               Message
    ----    ------     ----   ----               -------
    Normal  Scheduled  6m55s  default-scheduler  Successfully assigned default/nginx-azure-deployment-8659d8989-f8ct2 to master
    Normal  Pulling    6m54s  kubelet            Pulling image "nginx"
    Normal  Pulled     6m53s  kubelet            Successfully pulled image "nginx" in 1.35s (1.35s including waiting)
    Normal  Created    6m52s  kubelet            Created container nginx
    Normal  Started    6m52s  kubelet            Started container nginx
```
So far we have created a linkage between a pod and Persistent Volume.

## Quick Test

Let's do a quick test if the file in above PVC would be accessible from other pods.<br>
For that we could first make a file test.txt below.

```txt
$ kubectl get pod,pvc,pv
  NAME                                         READY   STATUS    RESTARTS   AGE
  pod/nginx-azure-deployment-8659d8989-f8ct2   1/1     Running   0          13m

  NAME                              STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
  persistentvolumeclaim/pvc-azure   Bound    pvc-2d558cbc-eace-4ed6-9322-f1c345871a25   1Gi        RWO            local-path     13m

  NAME                                                        CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM               STORAGECLASS   REASON   AGE
  persistentvolume/pvc-2d558cbc-eace-4ed6-9322-f1c345871a25   1Gi        RWO            Delete           Bound    default/pvc-azure   local-path              12m

$ kubectl exec -it nginx-azure-deployment-8659d8989-f8ct2 -- /bin/sh
  # cd /usr/share/nginx/html/web-app
  # pwd
  /usr/share/nginx/html/web-app
  # echo "Pineapples are not for pizza" > test.txt
  # ls
  test.txt
  # cat test.txt
  Pineapples are not for pizza
  # exit
```

We will then create a new pod below, go inside it and should find that test.txt file in pvc folder.

```txt
$ cat insp.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pvc-inspector
spec:
  containers:
  - image: busybox
    name: pvc-inspector
    command: ["tail"]
    args: ["-f", "/dev/null"]
    volumeMounts:
    - mountPath: /pvc
      name: pvc-mount
  volumes:
  - name: pvc-mount
    persistentVolumeClaim:
      claimName: pvc-azure

$ kubectl get pods
NAME                                     READY   STATUS    RESTARTS   AGE
nginx-azure-deployment-8659d8989-f8ct2   1/1     Running   0          32m
pvc-inspector                            1/1     Running   0          101s

$ kubectl exec -it pvc-inspector -- bin/sh
  / # 
  / # ls
  bin    dev    etc    home   lib    lib64  proc   pvc    root   sys    tmp    usr    var
  / # cd pvc
  /pvc # ls
  test.txt
  /pvc # 
```

# SUMMARY
Above exercise was done in K3S environment rather than Azure environment. Some feature may be different with Azure or any other cloud such as
1. Naming the Storage Class inside pvc.yaml would be limited, I found I only can use local-path
2. Unlike that we could only keep one pod at a time (the older pod shall be deleted) as defined in pvc.yaml like below, I could keep multiple pods in K3S. <br> ```accessModes: - ReadWriteOnce``` <br> 

That's how the storage was done dynamically in persistent volume, storage classes and persistent volume claim.

Kubernetes Storage with Persistent Volume Claims (PVC), Persistent Volumes (PV), and Storage Classes can provide more flexibility and control over storage provisioning and management in a Kubernetes cluster. It allows you to dynamically provision and manage storage resources based on the needs of your applications.

However, setting up and managing PVCs, PVs, and Storage Classes can be more complex and require additional configuration compared to using a Network File System (NFS) for storage. NFS provides a simpler and more straightforward way to share and access storage across multiple nodes in a cluster.

If you have a simple use case and don't require advanced storage management features, NFS might be a more straightforward and less cumbersome option. However, if you need more control and flexibility over storage provisioning and management, Kubernetes Storage with PVC, PV, and Storage Class can be a powerful solution.

| Feature                 | Kubernetes Storage (PVC, PV, Storage Class) | NFS                  |
|-------------------------|--------------------------------------------|----------------------|
| Scalability             | Highly scalable                             | Highly scalable      |
| Flexibility             | Supports various storage options             | Limited options      |
| Dynamic Provisioning    | Supports dynamic provisioning of storage     | Requires manual setup|
| Portability             | Can be easily moved between clusters         | Requires manual setup|
| Performance             | Depends on the underlying storage provider   | Depends on the network and server performance|
| Data Persistence        | Data is persisted even if pods are deleted   | Data is persisted    |
| Data Sharing            | Supports sharing data between pods           | Supports sharing data between systems|
| Backup and Recovery     | Can be easily backed up and recovered        | Requires manual setup|
| Cost                    | Costs depend on the underlying storage provider | Costs depend on the NFS server setup|
| Complexity              | Requires understanding of Kubernetes storage concepts | Requires understanding of NFS setup|
| High Availability       | Supports high availability through replication | Requires manual setup for high availability|
| Security                | Supports RBAC and access control             | Requires manual setup for access control|
| Monitoring and Metrics  | Can be integrated with Kubernetes monitoring tools | Requires separate monitoring setup|
| Fault Tolerance         | Supports fault tolerance through replication  | Requires manual setup for fault tolerance|