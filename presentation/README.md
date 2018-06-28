# DockerCon US 2018 Presentation - “Persistent and shared Storage”

The demonstration requires:

- NFS Share
- Docker EE
- UCP + 2 workers
- All nodes set to mixed

## Share Storage

Provision NFS server as you see fit, for the presentation I shared from my MacOS laptop.

Create a directory (in my case `/nfs` and export it)

**/etc/exports**

```
/nfs -maproot=root
```

**Enable NFS**

```
sudo nfsd start
```

## Provision initial application

Copy the contents of `1-Beginning.zip` into `/nfs`

## Choose your fighter (**SWARM** or **Kubernetes**)

### Swarm

The following `yaml` will deploy your example service using the shared and persistent storage. **ENSURE** you update the IP/hostname for the NFS server.

```
version: “3.3”

services:
  http:
    image: thebsdbox/dockercon:1.0
    deploy:
      replicas: 4
    ports:
      - “8443:443”
    volumes:
      - type: volume
        source: nfs
        target: /var/www/html 
        volume:
          nocopy: true
volumes:
  nfs:
    driver_opts:
      type: “nfs”
      o: “addr=172.16.49.1,nolock,soft,rw”
      device: “:/nfs”
```

You should now have four replicas that are running the `thebsdbox/dockercon:1.0` container, all of which will be reading and sharing content from the NFS share.

### Kubernetes

#### Define our Persistent Volume

We announce our persistent volume to the Kubernetes cluster with the following YAML spec.

**Note:** Ensure (as above) that the NFS server is the correct IP address. Other things to be aware of are things such as the `persistentVolumeReclaimPolicy` where we define the policy as `Retain` (ensuring our data isn’t removed after the pod is removed. Also the `accessModes` which specify that this volume is not only RW but also supports multiple nodes accessing it.

```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-pv
  labels: 
    volume-type: nfs
spec:
  capacity:
    storage: 10Gi 
  accessModes:
    - ReadWriteMany 
  persistentVolumeReclaimPolicy: Retain 
  nfs: 
    path: /nfs 
    server: 172.16.49.1
    readOnly: false
```

#### Claim some storage

The following YAML spec will attempt to claim some storage that matches it’s requirements, luckily that is exactly what we provided in the previous step.

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nfs-pvc  
spec:
  accessModes:
  - ReadWriteMany      
  resources:
     requests:
       storage: 1Gi
  selectors:
    matchLabels:
      volume-type: “nfs”  
```

#### Define our Replicaset

This consists of a Kubernetes deployment and a service that will expose out replicated application. Again it’s using the same container as above and makes use of the persistent volume claim.

Finally it exposes the replica set through a service, which ultimately will be the manager node and the port `32769`

```
apiVersion: apps/v1beta2
kind: Deployment
metadata:
  name: dockercon-deployment
spec:
  selector:
    matchLabels:
      app: dockercon
  replicas: 4
  template:
    metadata:
      labels:
        app: dockercon
    spec:
      containers:
      - name: dockercon
        image: thebsdbox/dockercon:1.0
        ports:
        - containerPort: 443
        volumeMounts:
          - name: nfsvol 
            mountPath: /var/www/html 
      volumes:
        - name: nfsvol
          persistentVolumeClaim:
            claimName: nfs-pvc
---  
apiVersion: v1
kind: Service
metadata:
  name: dockercon
  labels:
    app: dockercon
spec:
  type: NodePort
  ports:
    - port: 443
      nodePort: 32769
  selector:
    app: dockercon
```

### Remaining Demo

Connect to your `swarm` or `kubernetes` deployment and find the site deployed (well the under construction site). The final steps involve removing the contents of the `nfs` share and replacing them with `2-app.zip` which is now the correct site, but populated with #dockerselfies. 

### Final Dockercon site

Replace the contents of the `images` directory with the content of `3-Dockercon.zip` and now you have your persistent and shared storage and immutable containers. Kill pods/containers and feel free to scale up and down. Finally deploy the app using the other orchestrator and have them run side by side !