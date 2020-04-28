# Setup bitnami kafka charts on Centos using minikube.

## Minikube

### Setup minikube
Follow instruction to setup minikube
https://kubernetes.io/docs/tasks/tools/install-minikube/

For CentOs, this has proven use-full:
https://www.poftut.com/use-virt-manager-libvirt-normal-user-without-root-privileges-without-asking-password/

### Prepare minikube persistent volume folder
By default, all minikube vm data goes in ~/.minikube/ directory
This include all the Dynamically created Persistence Volume created by Kafka.
If you want minikube data to be stored at a different location, this includes disk space that will
be dynamically provisionned for Kafka, set minikube home a different directory:
```
export MINIKUBE_HOME=<local/path>   ex: /data/minikube
```

### Start minikube 
This start minikube with kvm virtualization,

It will use up to 48Gi from the host, 

It will use up to 16 cpus from the host

Allocate a ~400GB disk from within the path pointed by $MINIKUBE_HOME

```
minikube start 
  --driver=kvm2 \  
  --memory 48Gi \
  --disk-size=400GB \
  --cpus=16
```


### Check the minikube instance

cd into the folder showed below and check the internal raw disk that was created.

```
cd $MINIKUBE_HOME/.minikube/machines/minikube
ls -lh
total 52G
-rw-------. 1 qemu     qemu     175M Apr 27 09:56 boot2docker.iso
-rw-------. 1 fardoche fardoche 2.8K Apr 28 11:57 config.json
-rw-------. 1 fardoche fardoche 1.7K Apr 27 09:56 id_rsa
-rw-------. 1 fardoche fardoche  381 Apr 27 09:56 id_rsa.pub
-rw-r--r--. 1 qemu     qemu     382G Apr 28 16:02 minikube.rawdisk
```

Check minikube allocated memory and cpus

```
minikube ssh

cat /proc/meminfo 
MemTotal:       47152424 kB

lscpu 
Architecture:                    x86_64
CPU op-mode(s):                  32-bit, 64-bit
Byte Order:                      Little Endian
Address sizes:                   48 bits physical, 48 bits virtual
CPU(s):                          16

df -h
Filesystem      Size  Used Avail Use% Mounted on
...
/dev/vda1       333G  2.8G  311G   1% /mnt/vda1
```


### Export minikube IP
This will be useful for next steps...

```
export MINIKUBE_IP=$(minikube ip)
```

### Configure firewall 

Allow minikube to connect to host (if you do extra NFS share)

```
sudo firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="'$MINIKUBE_IP'/24" accept'
```

## Kafka Charts
### Start Kafka Instance bundled with Zookeeper, external port configured with NodePort

```
helm install kafka bitnami/kafka \
  --set zookeeper.enabled=true \
  --set replicaCount=3 \
  --set externalAccess.enabled=true \
  --set externalAccess.service.type=NodePort \
  --set externalAccess.service.nodePorts[0]='30090' \
  --set externalAccess.service.nodePorts[1]='30091' \
  --set externalAccess.service.nodePorts[2]='30092' \
  --set externalAccess.service.domain=$MINIKUBE_IP
```

### Check services running
```
>kubectl get all
NAME                    READY   STATUS    RESTARTS   AGE
pod/kafka-0             1/1     Running   1          48s
pod/kafka-1             1/1     Running   0          17s
pod/kafka-2             1/1     Running   0          17s
pod/kafka-zookeeper-0   1/1     Running   0          50s

NAME                               TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
service/kafka                      ClusterIP   10.101.110.207   <none>        9092/TCP                     50s
service/kafka-0-external           NodePort    10.98.159.215    <none>        19092:30090/TCP              50s
service/kafka-1-external           NodePort    10.99.233.61     <none>        19092:30091/TCP              50s
service/kafka-2-external           NodePort    10.100.28.102    <none>        19092:30092/TCP              50s
service/kafka-headless             ClusterIP   None             <none>        9092/TCP                     51s
service/kafka-zookeeper            ClusterIP   10.108.5.96      <none>        2181/TCP,2888/TCP,3888/TCP   50s
service/kafka-zookeeper-headless   ClusterIP   None             <none>        2181/TCP,2888/TCP,3888/TCP   51s
service/kubernetes                 ClusterIP   10.96.0.1        <none>        443/TCP                      29h
```


### Test Kafka Access From Host
Install kafka client tools on your host and try these commands, or run your own consumer/producer application.
```
kafka-topics.sh --create --bootstrap-server $MINIKUBE_IP:30090 --replication-factor 1 --partitions 10 --topic test
kafka-topics.sh --list  -bootstrap-server $MINIKUBE_IP:30090
kafka-console-producer.sh --broker-list $MINIKUBE_IP:30090 --topic test
>1
>2
>4
ctrl-c
kafka-console-consumer.sh --bootstrap-server $MINIKUBE_IP:30090 --topic test --from-beginning
4
1
2
```




## Mounts and minikube

