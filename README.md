# Minecraft Server

Resources and instructions related to deploying and managing a minecraft server. There are two
types of server:

- Minecraft for the Java Edition. This can be connected to using the Java client.
- Bedrock Dedicated Server. This can be connected to with the XBox and Mobile devices.

The following instructions are all related to managing the Bedrock server.

## Managing a local Bedrock server

You can easily get a Minecraft Bedrock server up and running on your own laptop/desktop by using
[a containerised version of the software](https://hub.docker.com/r/itzg/minecraft-bedrock-server)
that greatly simplifies the installation.

### Deploying the server

The following docker command runs a world in `creative` mode as a background process:

```sh
docker run -v mc:/data -it --rm --name mc -d \
  -e EULA=TRUE -e GAMEMODE=creative -e ALLOW_CHEATS=true \
  -p 19132:19132/udp itzg/minecraft-bedrock-server
```

A volume is mounted for the `/data` directory so that you can restart the process without losing
your game. The container bootstraps when it starts, downloading packages, which may take some time.
This can be watched using:

```sh
docker logs -f mc
```

When finished, you can connect to your server with a mobile device on
`{your computer's IP address}:19132` and start playing!

### Backing up your worlds data

If you wish to back up a game in it's current state, it's the `/data/worlds` directory that
contains the world data. The follow commands show you how to do this:

```sh
export TIMESTAMP=$(date "+%Y%m%d-%H%M")
# disconnect the server to stop any changes while you are backing up
docker network disconnect bridge mc
# zip the data to /tmp
docker exec mc zip -r /tmp/mc.$TIMESTAMP.zip /data/worlds
# copy the zip to your backup location
docker cp mc:/tmp/mc.$TIMESTAMP.zip /path/to/backup/location
# reconnect the server to stop any changes while you are backing up
docker network disconnect bridge mc
```

## Managing a Bedrock server on a GCP Kubernetes Cluster

Running a local server might be fine for you. However,  if you want your server to be more stable
and more widely available, the following sections describe how you can run you server in the cloud -
specifically on a [GCP Kubernetes Cluster](https://cloud.google.com/kubernetes-engine).

### Deploying the server

To deploy the server you need to complete the following three steps.

1. Create [a GCP project](https://cloud.google.com/storage/docs/projects) to contain your cluster.
A container has been developed to help with this and instructions for it's use can be found in
[`create-gcp-project\README.md`](https://github.com/msb/minecraft/blob/master/create-gcp-project/README.md).
~~TODO: update when available on Docker Hub~~

2. Create the cluster. A terraform project has been created to simplify this and instructions for
   it's use can be found in
   [`tiny-cluster\README.md`](https://github.com/msb/minecraft/blob/master/tiny-cluster/README.md).
   ~~TODO update when available on Docker Hub~~

3. Deploy minecraft to the cluster. If you have completed the last two steps, you should have the
necessary containers/variables locally to run:

```sh
# TODO capture this in a script
docker run -it --rm --env-file tiny-cluster/kube.env \
  --volumes-from=$PROJECT_CONTAINER --volume=$PWD:/minecraft \
  google/cloud-sdk bash
```

This command will put you in a container where you can manage your cluster by running
[`kubectl`](https://kubernetes.io/docs/reference/kubectl/kubectl/) commands. Now deploy Minecraft:

```sh
kubectl apply -f /minecraft/kubernetes.yml
```

`kubernetes.yml` defines how the server will be deployed to the cluster. You can make changes to
the file (eg. change the game mode), re-run the command and Kubernetes will re-deploy the changes
and re-start the server for you. This file is based on the 
[itzg/docker-minecraft-bedrock-server example](https://github.com/itzg/docker-minecraft-bedrock-server/blob/master/examples/kubernetes.yml). You can check the progress of the installation with:

```sh
kubectl logs -f deployment/bds
```

When complete you can read the IP address to connect on using:

```sh
kubectl get services
```

.. and connecting to the game using the `EXTERNAL-IP` listed (the port is still 19132).

Subsequent instructions will require you to set a variable with 
[the server's pod](https://kubernetes.io/docs/concepts/workloads/pods/pod/):

```sh
POD=$(kubectl get pod --field-selector=status.phase=Running | sed 's/ .*//' | tail -1)
```

### Backing up your worlds data

Doing a backup roughly the same as above but using `kubectl`. To ensure consistent data, attach to
the Minecraft process using `kubectl attach -it $POD` and run the command 
[`save hold`](https://minecraft.gamepedia.com/Commands/save). Then in another window run:

```sh
TIMESTAMP=$(date "+%Y%m%d-%H%M")
kubectl exec -it $POD -- zip -r /tmp/mc.$TIMESTAMP.zip /data/worlds
kubectl cp $POD:/tmp/mc.$TIMESTAMP.zip /minecraft/mc.$TIMESTAMP.zip
```

When complete, return to the 1st window and type 
[`save resume`](https://minecraft.gamepedia.com/Commands/save). When you disconnect from the
process, make sure you use ^p^q. Otherwise you may kill the process (although I think K8s should
just restart the pod).

### Restoring a world

To be sure of no data corruption when restoring, I would suggest the following strategy:

- Pick a new LEVEL_NAME (eg 'creative-2') and update `kubernetes.yaml`
- Unzip your selected backup in your current dir and copy it up to your server with the new 
  LEVEL_NAME: `kubectl cp /minecraft/data/worlds/creative-1/ $POD:/data/worlds/creative-2`
- The run `kubectl apply -f /minecraft/kubernetes.yml` to restart the server with the new
  LEVEL_NAME

### Adding an operator to `permissions.json`

The server defined in `kubernetes.yaml` has cheats enabled. But to use them you need to be an
operator. To make a player a permanent operator you can added them to the `permissions.json` file.
A `permissions.json.in` file is provided to show you the format required. You identify a player by 
his `xuid` which can be read from the logs when they enter the game. Once you have the `xuid`, copy
the `permissions.json.in` file to `permissions.json` and update it. Then:

```sh
kubectl cp $POD:permissions.json $POD:/data/
# restart the server
kubectl rollout restart deployment bds
```
