## How to Setup Kubernetes Loadbalancer on Baremetal

NOTE: [docker](https://docs.docker.com/engine/install/ubuntu/) is required to be installed on the host.
### Install Docker on Ubuntu 18.04
```
sudo apt update && sudo apt upgrade
```
```
sudo apt-get install curl apt-transport-https ca-certificates software-properties-common
```
```
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
```
```
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
```
```
sudo apt update
```
```
sudo apt install docker-ce
```
```
sudo systemctl status docker
```

### To run docker without sudo command
```
sudo usermod -aG docker $USER
```
```
newgrp docker
```

### Confirm docker is running properly
```
docker run hello-world
```
This will print to console an informational message.

For macOS and windows, install [docker-desktop](https://www.docker.com/products/docker-desktop).

### Installation on Linux
We are going to be using [kind](https://github.com/kubernetes-sigs/kind) to create our local kubernetes cluster. You can get kind
```
wget https://github.com/kubernetes-sigs/kind/releases/download/v0.9.0/kind-linux-amd64
```

For more information on kind, visit: https://kind.sigs.k8s.io/  

Create executable with 
```
chmod +x kind-linux-amd64
```  

Run: `sudo mv kind-linux-amd64 /usr/local/bin/kind`  

Confirm installation of kind, `which kind` and `kind version`. This will list kind version information.

### Installation on MacOS
```
brew install kind
```

## Create cluster
```
kind create cluster
```  

NOTE: If you have go (1.11+) and docker installed GO111MODULE="on" `go get sigs.k8s.io/kind@v0.9.0 && kind create cluster` is all you need!  

confirm cluster is running `kubectl cluster-info`  

## Create nginx deployment and service
```
kubectl create deploy nginx --image nginx
```
```
kubectl expose deploy nginx --port 80 --type LoadBalancer
```  

## MetalLB
![service-pending](https://github.com/learningdollars/kubernetes-loadbalancer-localhost-bare-metal/blob/main/images/kind-pending.PNG)  

On bear metal servers, services of type LoadBalancer will remain in pending state. This can be resolved by using [metallb](https://metallb.universe.tf/installation/). Install metallb by manifest according to the docs and setup the Layer2 [configuration](https://metallb.universe.tf/configuration/) or by running:  
```
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.9.4/manifests/namespace.yaml
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.9.4/manifests/metallb.yaml
# On first install only
kubectl create secret generic -n metallb-system memberlist --from-literal=secretkey="$(openssl rand -base64 128)"
```

Get usable IP address `kubectl get nodes -o wide` and pay attention to the Internal IP range.  

After taking note of the internal IP range, select a subset of the available IPs that will be used by the services of the LoadBalancer type. For example, the internal IP is in range `172.19.x.x`. Take the range `172.19.255.1-172.19.255.255`. This will be the external IP range for the external facing services of type LoadBalancer.  

Replace the IP range in `metallb/config.yaml` and apply the configMap with  
```
kubectl apply -f config.yaml
```

After successfully installing metallb, the external service will no longer be in a pending state. 
Note: You may have to delete and recreate the service if it remains in pending state.  

![service-success](https://github.com/learningdollars/kubernetes-loadbalancer-localhost-bare-metal/blob/main/images/kind-ip-loadbalancer.PNG)  

### Port forward
```
kubectl port-forward service/nginx 5000:80
```  
This will listen on port 5000 and forward port 80 on service ngnix. 

Visit `http://localhost:5000` and the nginx homepage is visible.

![nginx](https://github.com/learningdollars/kubernetes-loadbalancer-localhost-bare-metal/blob/main/images/kind-ngnix.PNG)
