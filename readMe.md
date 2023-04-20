   ## Setup gcloud
   ---

    gcloud init
    gcloud auth login
    gcloud config set compute/region us-west1
    gcloud config set compute/zone us-west1-c


    gcloud compute networks create dream-kube --subnet-mode custom

    gcloud compute networks subnets create dream-kube \
    --network dream-kube \
    --range 10.240.0.0/24

    gcloud compute firewall-rules create dream-kube-allow-internal \
    --allow tcp,udp,icmp,ipip \
    --network dream-kube \
    --source-ranges 10.240.0.0/24


    gcloud compute firewall-rules create dream-kube-allow-external \
    --allow tcp:22,tcp:6443,icmp \
    --network dream-kube \
    --source-ranges 0.0.0.0/0

    gcloud compute instances create controller-1 \
        --async \
        --boot-disk-size 200GB \
        --can-ip-forward \
        --image-family ubuntu-2004-lts \
        --image-project ubuntu-os-cloud \
        --machine-type e2-standard-2 \
        --private-network-ip 10.240.0.11 \
        --scopes compute-rw,storage-ro,service-management,service-control,logging-write,monitoring \
        --subnet dream-kube \
        --zone us-west1-c \
        --tags dream-kube,controller

    gcloud compute instances create worker-1 \
        --async \
        --boot-disk-size 200GB \
        --can-ip-forward \
        --image-family ubuntu-2004-lts \
        --image-project ubuntu-os-cloud \
        --machine-type e2-standard-2 \
        --metadata pod-cidr=10.200.1.0/24 \
        --private-network-ip 10.240.0.21 \
        --scopes compute-rw,storage-ro,service-management,service-control,logging-write,monitoring \
        --subnet dream-kube \
        --zone us-west1-c \
        --tags dream-kube,worker


    gcloud compute instances create worker-2 \
        --async \
        --boot-disk-size 200GB \
        --can-ip-forward \
        --image-family ubuntu-2004-lts \
        --image-project ubuntu-os-cloud \
        --machine-type e2-standard-2 \
        --metadata pod-cidr=10.200.1.0/24 \
        --private-network-ip 10.240.0.22 \
        --scopes compute-rw,storage-ro,service-management,service-control,logging-write,monitoring \
        --subnet dream-kube \
        --zone us-west1-c \
        --tags dream-kube,worker

## Instaling Docker to all instances
    sudo apt-get update 
    sudo apt-get install \
        ca-certificates \
        curl \
        gnupg

    sudo mkdir -m 0755 -p /etc/apt/keyrings
    curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

    echo \
    "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
    "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
    sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

    sudo apt-get update
    sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
    sudo docker run hello-world

## Kubeadm, kubelet, kubectl

    sudo apt-get update
    sudo apt-get install -y apt-transport-https ca-certificates curl
    sudo curl -fsSLo /etc/apt/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
    echo "deb [signed-by=/etc/apt/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
    sudo apt-get update
    sudo apt-get install -y kubelet kubeadm kubectl
    sudo apt-mark hold kubelet kubeadm kubectl

### If you face problem

    1. echo \
    "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
    "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
    sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
    2. Remove the old containerd: sudo apt remove containerd
    3. Update repository data and install the new containerd:sudo  apt update,sudo apt install containerd.io
    4. Remove the installed default config file: sudo rm /etc/containerd/config.toml
    5. Restart containerd:sudo systemctl restart containerd. Set up the Docker repository as described in https://docs.docker.com/engine/install/ubuntu/#set-up-the-repository


---
    mkdir -p $HOME/.kube
    sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    sudo chown $(id -u):$(id -g) $HOME/.kube/config


## Join worker nodes

    sudo kubeadm join 10.240.0.11:6443 --token <token> \
            --discovery-token-ca-cert-hash <discovery-token>

## Install Calico
---
    kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.25.0/manifests/tigera-operator.yaml

    kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.25.0/manifests/custom-resources.yaml

    watch kubectl get pods -n calico-system

## Pushing image to gcr
---

    docker tag dream_java_project gcr.io/<project_id>/dream:1.0

    docker push gcr.io/<project-id>/dream:1.0

---

## Deployment

    kubectl apply -f manifast/deployment.yaml

---


## Setting up load balancer

    gcloud compute firewall-rules create fw-allow-network-lb-health-checks \
        --network=dream-kube\
        --action=ALLOW \
        --direction=INGRESS \
        --source-ranges=0.0.0.0/0\
        --target-tags=allow-network-lb-health-checks \
        --rules=tcp
        
    gcloud compute health-checks create https k8s-controller-hc --check-interval=5 \
        --enable-logging \
        --request-path=/healthz \
        --port=6443 \
        --region=us-west1
	
# crate a backend service
    gcloud compute backend-services create k8s-service \
        --protocol TCP \
        --health-checks k8s-controller-hc \
        --health-checks-region us-west1 \
        --region us-west1
        
    gcloud compute backend-services add-backend k8s-service \
        --instance-group dream-instance-group-1 \
        --instance-group-zone us-west1-c \
        --region us-west1
        
    gcloud compute addresses create k8s-lb --region us-west1

    gcloud compute forwarding-rules create k8s-forwarding-rule \
        --load-balancing-scheme external \
        --region us-west1 \
        --ports 31000 \
        --address k8s-lb \
        --backend-service k8s-service
        
    -------


    kubectl patch svc dream-service -n kubespace-p '{"spec": {"type": "NodePort", "externalIPs":["<public_ip>"]}}'

---

## Setup Jenkins

    Follow https://cloud.google.com/architecture/using-jenkins-for-distributed-builds-on-compute-engine

## Argocd setup

Follow https://argo-cd.readthedocs.io/en/stable/getting_started/
    
### To access ArgoCD
    kubectl patch svc argocd-server -n argo-cd -p '{"spec": {"type": "NodePort", "externalIPs":["<public_ip>"]}}'
    ---
    gcloud compute forwarding-rules create k8s-forwarding-rule3 \
        --load-balancing-scheme external \
        --region us-west1 \
        --ports 8082\
        --address k8s-lb \
        --backend-service k8s-service

---

Birde portu 80'den 8082'iye çektim 

    kubectl edit svc argocd-server -n argocd 
---

# Monitoring

* Mounting Disk  
First from gcloud interface, attach disk.

        sudo mkfs.ext4 -m 0 -E lazy_itable_init=0,lazy_journal_init=0,discard /dev/sdb
        sudo mkdir -p /mnt/disks/disk1
        sudo mount -o discard,defaults /dev/sdb /mnt/disks/disk1
        sudo chmod a+w /mnt/disks/disk1
        add this to /etc/fstab
        UUID=643eb997-1db2-452c-b955-459777eb17fa /mnt/disks/ ext4 discard,defaults 0 2

* Create pv and storageclass

        kubectl apply -f pv.yaml 
        kubectl apply -f storageclass.yaml

* To release pv

kubectl patch pv prometheus-pv -p '{"spec":{"claimRef": null}}'
kubectl patch pv prometheus-pv2 -p '{"spec":{"claimRef": null}}'


* Getting Kubernetes Metrics Server

        helm repo add metrics-server https://kubernetes-sigs.github.io/metrics-server/

        helm install metrics-server metrics-server/metrics-server --set args='{--kubelet-insecure-tls}'

* Helm install prometheus
    
        helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
        helm repo update
    ---

        helm install prometheus prometheus-community/prometheus \
        --namespace prometheus \
        --set alertmanager.persistence.storageClass="prometheusstorage" \
        --set server.persistentVolume.storageClass="prometheusstorage" 

* Accsesing prometheus

        kubectl patch svc prometheus-server -n prometheus -p '{"spec": {"type": "NodePort", "externalIPs":["<public_ip>"]}}'
       
       kubectl edit svc prometheus-server 
       (portu 9100 çektim)

        gcloud compute forwarding-rules create k8s-forwarding-rule4 \
        --load-balancing-scheme external \
        --region us-west1 \
        --ports 9100\
        --address k8s-lb \
        --backend-service k8s-service

* Helm Install Grafana

        helm repo add grafana https://grafana.github.io/helm-charts 
        helm repo update 
        kubectl create namespace grafana
        helm install grafana grafana/grafana \
            --namespace grafana \
            --set adminPassword='EKS!sAWSome' \
            --values prometheus-datasource.yaml \
            --set service.type=NodePort 


* Accsesing grafana
       
       kubectl edit svc prometheus-server 
       (portu 9200 çektim)

       kubectl patch svc grafana -n grafana -p '{"spec": {"type": "NodePort", "externalIPs":["<public_ip>"]}}'

        gcloud compute forwarding-rules create k8s-forwarding-rule5 \
        --load-balancing-scheme external \
        --region us-west1 \
        --ports 9200\
        --address k8s-lb \
        --backend-service k8s-service


