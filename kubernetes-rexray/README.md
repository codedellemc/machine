# Docker Machine with Kubernetes and REX-Ray

This repo can help you quickly get Kubernetes up and running with persistent
volumes. All of the configuration is done for you, so please take a few minutes
and enjoy the demo.

The external volume capability here focuses on VirtualBox and is provided
through [REX-Ray](https://github.com/emccode/rexray). Other drivers from REX-Ray
should be configured in a typical Kubernetes fashion. This is currently in
Alpha state!

## Requirements

* VirtualBox 5.0.10+
* Docker Toolbox 1.10+
* Internet connection

## Pre-requisites

1. [Install Docker Toolbox](https://www.docker.com/products/docker-toolbox)

2. Disable authentication for VirtualBox and start the SOAP Web Service.

  ```bash
  VBoxManage setproperty websrvauthlibrary null
  /Applications/VirtualBox.app/Contents/MacOS/vboxwebsrv -H 0.0.0.0 -v
  ```

## Start Kubernetes

1. Create the Docker VM

  ```bash
  docker-machine create --driver=virtualbox k8
  eval $(docker-machine env k8)
  ```

2. (optional) This step is required if you need to create new volumes manually to attach to pods. It is not required if you are going to leverage Persistent Volume Claims or if you already have volumes created.

  ```bash
  docker-machine ssh k8 "curl -sSL https://dl.bintray.com/emccode/rexray/install | sh -s staged"
  ```

3. Create a REX-Ray configuration file. *There are two fields that are cirtical to change!*  Make sure to present a valid path by replacing `user` under the `volumePath` parameter. Also make sure to update the `localMachineOrId` with the name of the docker-machine, in this case we deployed a machine with `k8` so no change was necessary.

   ```bash   
    docker-machine ssh k8 "sudo mkdir -p /etc/rexray && sudo tee -a /etc/rexray/config.yml << EOF
    rexray:
      storageDrivers:
      - virtualbox
      modules:
        default-docker:
          rexray:
            volume:
              mount:
                preempt: true
        kubernetes:
          type: docker
          rexray:
            volume:
              mount:
                preempt: true
                ignoreUsedCount: true
    virtualbox:
      endpoint: http://10.0.2.2:18083
      tls: false
      volumePath: /Users/user/VirtualBox Volumes
      controllerName: SATA
      localMachineOrId: k8"
    ```

3. Start the `kubelet-runner` container. This is responsible for launching all of the other necessary Kubernetes services as containers.

    ```bash
    docker run \
      --volume=/:/rootfs:ro \
      --volume=/sys:/sys:ro \
      --volume=/dev:/dev \
      --volume=/var/lib/docker/:/var/lib/docker:rw \
      --volume=/var/lib/kubelet/:/var/lib/kubelet:rw \
      --volume=/var/run:/var/run:rw \
      --volume=/etc/rexray:/etc/rexray:ro \
      --net=host \
      --pid=host \
      --privileged=true \
      -d emccode/hyperkube-amd64:v1.2.0-alpha.8 \
      /kubelet-runner.sh \
        --hostname-override="127.0.0.1" \
        --address="0.0.0.0" \
        --api-servers=http://localhost:8080 \
        --config=etc/kubernetes/manifests \
        --cluster-dns=10.0.0.10 \
        --cluster-domain=cluster.local \
        --allow-privileged=true --v=4
    ```

4. Launch a container that has the `kubectl` tool running. This can be done remotely as well.

    ```bash
    docker run -ti --rm --net=host -v /var/lib/rexray:/var/lib/rexray emccode/hyperkube-amd64:v1.2.0-alpha.8 /bin/bash
    ```

5. Wait for Kubernetes to be ready. It will return connection refused messages until it is ready.

    ```bash
    $ ./kubectl get nodes
    NAME        STATUS    AGE
    127.0.0.1   Ready     0s
    ```

## Volumes
In Kubernetes there are two types of volumes. The first would be considered just normal volumes that are implicitly associated with pods as they get created. The second would be considered `Persistent Volumes` which involve a set of steps to create available claims to certain types of storage while allowing a consumer to request volumes that get matched to the available claims.

We will go through the following two major sections now.

1. Volumes with Pods
2. Persistent Volumes
 - Persistent Volume Claims (alpha)
 - Persistent Volume Claims with Pods (alpha)

### Volumes With Pods
This first method assumes that you have created volumes outside of Kubernetes. They are defined by name in the `podvol.yml` file and the `VolumeName` parameter.

1. (optional if volumes already exist) Create a volume with REX-Ray

    ```bash
    (exit from container)
    docker-machine ssh k8 sudo rexray volume create --volumename=test123 --size=1
    ```

2. Launch a container that has the `kubectl` tool running.

    ```bash
    docker run -ti --rm --net=host -v /var/lib/rexray:/var/lib/rexray emccode/hyperkube-amd64:v1.2.0-alpha.8 /bin/bash
    ```

3. Create your first pod with a volume attached.

    ```bash
    ./kubectl create -f podvol.yml
    ```

4. You can check the status of your pod by running `./kubectl describe pod mypod1`. This should return the configuration along with a list of events at the bottom. If all is good then your pod should refernce  `Started`. You can verify separately by looking at the existing mounts for the VM.

    ```bash
    (exit from container)
    docker-machine ssh k8 cat /proc/mounts | grep test123
    /dev/sdm /var/lib/rexray/volumes/test123 ext4 rw,relatime,data=ordered 0 0
    ```

### Persistent Volume Claims
The second method involves steps to enable dynamic `Persistent Volumes` through `Persistent Volume Claims`.

1. Launch a container that has the `kubectl` tool running.

    ```bash
    docker run -ti --rm --net=host -v /var/lib/rexray:/var/lib/rexray emccode/hyperkube-amd64:v1.2.0-alpha.8 /bin/bash
    ```

2. First we need to create a claim. This is captured in the `myclaim.yml` file
and includes a storage class attribute where we defined `volume.alpha.kubernetes.io/storage-class: kubernetes`.
The `kubernetes` part of this definition is where we can define the different types of storage that may
be available. This relates to `modules` in REX-Ray.

    Create the claim with the following command.

    ```bash
    ./kubectl create -f myclaim.yml
    ```

3. Check on the status of the claim. It should say `Bound` with an associated
`Volume` parameter populated with a dynamic name. This claim is now ready to be
used for a pod.

    ```bash
    ./kubectl describe pvc myclaim
    Name:		myclaim
    Namespace:	default
    Status:		Bound
    Volume:		rexray-lsc0z
    Labels:		<none>
    Capacity:	1Gi
    Access Modes:	RWO
    ```

4. You can step it a step further and look at the volume that was created.
Notice how the information specific to the storage platform is automatically
populated.

    ```bash
    ./kubectl describe pv rexray-lsc0z
    Name:		rexray-lsc0z
    Labels:		<none>
    Status:		Bound
    Claim:		default/myclaim
    Reclaim Policy:	Delete
    Access Modes:	RWO
    Capacity:	1Gi
    Message:
    Source:
        Type:		RexRay (a Persistent Disk resource in REXRay)
        VolumeName:		kubernetes-dynamic-rexray-lsc0z
        VolumeID:		f2409271-ef2a-4633-858c-7d45bd5d39e4
        Module:		kubernetes
        StorageDriver:	virtualbox
    ```


### Persistent Volume Claims with Pods
Finally the last step is to actually consume a claim with a pod.

1. Create the pod usingt the following command.

    ```bash
    ./kubectl create -f podclaim.yml
    ```

2. Go ahead and describe the status. Notice how the container is using the claim.
This example is using Postgres, so it may take more time for the container to
download and become ready.

    ```bash
    ./kubectl describe pod mypod2
    ...
    Volumes:
  postgres-data:
    Type:	PersistentVolumeClaim (a reference to a PersistentVolumeClaim in the same namespace)
    ClaimName:	myclaim
    ReadOnly:	false
    ```

3. Optionally you can verify the volume has mounted and data exists. Find the
volume associated as defined below by `rexray-lsc0z`.

    ```bash
    ./kubectl describe pvc myclaim
    Name:		myclaim
    Namespace:	default
    Status:		Bound
    Volume:		rexray-lsc0z
    ```

4. Look at the existing mounts and look for that volume name in the mounts. There
should be two mounts. The first listed is the direct mount from REX-Ray and the second
is a private mount for the container.

    ```bash
    (exit from container)
    docker-machine ssh k8 cat /proc/mounts | grep rexray-lsc0z
    ...
    /dev/sdp /var/lib/rexray/volumes/kubernetes-dynamic-rexray-lsc0z ext4 rw,relatime,data=ordered 0 0
    /dev/sdp /var/lib/kubelet/pods/13debc7a-e1a5-11e5-81ca-32cc5a798e6a/volumes/kubernetes.io~rexray/kubernetes-dynamic-rexray-lsc0z ext4 rw,relatime,data=ordered 0 0
    ```


# Done!

That's it!
