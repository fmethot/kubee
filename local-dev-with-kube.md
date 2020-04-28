# Centos guide to have bitnami kafka helm chart running locally using minikube

# Setup minikube
Follow instruction to setup minikube

For CentOs, this has proven use-full:
https://www.poftut.com/use-virt-manager-libvirt-normal-user-without-root-privileges-without-asking-password/

# Prepare minikube persistent volume folder
Create a directory on your host where minikube data will be store. This includes disk space that will
be dynamically provisionned for Kafka.
(Default directory in is home user dir, if you keep default, make sure there is enough disk space.)
```
mkdir /data/minikube
```

Set minikube home  to that folder
```
export MINIKUBE_HOME=/data/minikube
```

# start minikube 
minikube start 
  --driver=kvm2 \
  --mount-string="/data/kubedata:/data" --mount \
  --memory 48Gi \
  --disk-size=400GB \
  --cpus=16

# export minikube IP
export MINIKUBE_IP=$(minikube ip)

# setup firewall 
Allow minikube to connect from host
```
sudo firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="'$MINIKUBE_IP'/24" accept'
```



