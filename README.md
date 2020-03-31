# k8s rebrain cource
### Get started

1. Install Kubernetes & WeavNet plug-in by following these steps.
You have to install Kubernetes on all the machines (VMs/instances) used in the environment

Installing Docker
``` sudo su apt-get update apt-get install -y docker.io

```

IMPORTANT!! If you have proxies on your VMs you need to configure the proxies in docker by following (https://docs.docker.com/config/daemon/systemd/) under the HTTP/HTTPS section.

Restart Docker systemctl daemon-reload systemctl restart docker

Installing kubelet, kubectl, and kubeadm
``` apt-get update && apt-get install -y apt-transport-https curl

curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add - cat <<EOF >/etc/apt/sources.list.d/kubernetes.list deb http://apt.kubernetes.io/ kubernetes-xenial main EOF

apt-get update apt-get install -y kubelet kubeadm kubectl ```

To install a specific version of K8s run apt-get install -y kubelet=<vserion> kubeadm=<vserion> kubectl=<vserion>

After intstall Kubernetes, you need to set it up.
If the interface you use for Kubernetes management traffic (for example, the IP address used for kubeadm join) is not the one that contains the default route out of the host, you need to specify the management node IP address in the Kubelet config file. Add the following line to (/etc/systemd/system/kubelet.service.d/10-kubeadm.conf): Environment="KUBELET_EXTRA_ARGS=--fail-swap-on=false --feature-gates HugePages=false --node-ip=<node-ip-address>"

For Example: If the VMs IP is 10.10.10.10 the above line would be like Environment="KUBELET_EXTRA_ARGS=--fail-swap-on=false --feature-gates HugePages=false --node-ip=10.10.10.10" Restart kubelet: systemctl daemon-reload systemctl restart kubelet initialize the master node by running the following code NOT as a root change the IP address accordingly sudo swapoff -a sudo kubeadm init --token-ttl 0 --apiserver-advertise-address=10.10.10.10 If you ran into errors while running kubeadm command, just use the following steps to reset kubeadm and do kubeadm init again rm -rf $HOME/.kube sudo kubeadm reset After initializing the master run the following commands mkdir -p $HOME/.kube sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config sudo chown $(id -u):$(id -g) $HOME/.kube/config Then add the K8s network plug-in (I choose WeaveNet) kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')" After running the command kubeadm init ... , you will have a command to join the worker nodes. it looks like this Run this command if you ran into swap issue sudo swapoff -a

Joining the worker node/nodes
kubeadm join 10.10.10.10:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash> After joining the workers to the master, run this command on the master to check if node are connected or not kubectl get node The output should be similar to this NAME STATUS ROLES AGE VERSION mcent1 Ready master 10m v1.10.4 mcent2 Ready <none> 2m v1.10.4

If you want to have kubectl autocompele add this line in .bashrc source <(kubectl completion bash), then reboot the master

Just to test if Kubernetes is running smoothly run the following command to deploy nginx containers kubectl run nginx --image=nginx --replicas=4 Then run kubectl get pods -owide All four containers should show the status of Running

After that delete the deployment kubectl delete deployment nginx

The cleaning up of Kubernetes
Remode the network plug-in ''' kubectl delete -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')" ''' drain the node then delete it kubectl drain <node-name> --delete-local-data --force --ignore-daemonsets kubectl delete <node-name> Then remove .kube and reset kubeadm rm -rf $HOME/.kube sudo kubeadm reset

2. Kubernetes APIs (https://kubernetes.io/docs/reference/)
To start using Kubernetes APIs, you need to turn on K8s reverse proxy, by running the following
kubectl proxy --port 8081 --address=<K8s_IP_Address> --accept-hosts=^* &

--accept-hosts=^* flag means that accept any request from any user. Ideally you should include only specific hosts (like yourself and/or the GUI).

The list of K8s APIs to use after turning on the reverse proxy (the return value would be in JSON fromat):
<K8s_IP_Address>:8081/api to get the API version.
<K8s_IP_Address>:8081/api/v1/namespaces to get all the namespaces in the K8s cluster.
<K8s_IP_Address>:8081/api/v1/pods to get all the pods in the K8s cluster (pods' IPs, names, where does it exist, etc...).
<K8s_IP_Address>:8081/api/v1/services to get all the services in the K8s cluster (ports exposed by K8s).
<K8s_IP_Address>:8081/api/v1/namespaces/<namespace>/pods to get the pods that exists in that nemespace (replace <namespace> with the namespace needed)
<K8s_IP_Address>:8081/apis/extensions/v1beta1/namespaces/<namespace>/deployments to get all the pods and containers' info (port #, pod name, # of conainers per pod, containers' specs)
Uses of these APIs:
Extract information out of K8s as JSON reply.
Navigate through the JSON reply and extract pods' names, IPs, etc...
Use the APIs for the GUI.
Examples (need to run sudo apt install jq):
curl http://10.195.77.197:8081/api/v1/namespaces/<namespace>/pods | jq -r '.items[] | .spec.containers[].name' to get the conatiners' names in the <namespace> namespace.
curl http://10.195.77.197:8081/api/v1/namespaces/<namespace>/pods | jq -r '.items[] | .spec.containers[].name+ " ===> "+ .status.podIP' to get the conatiners' names in the default namespace + pods' IP addresses.
To turn off the reverse proxy, run the following
ps -ef | grep kubectl the output should be similar to this localad+ 20315 19702 0 16:40 pts/0 00:00:00 kubectl proxy --port 8081 --address=<K8s_IP_Address> --accept-hosts=^* localad+ 20424 19702 0 16:41 pts/0 00:00:00 grep --color=auto kubectl copy the PID which in this case 20315 and kill it kill -9 20315

3. Installing Istio, I used Istio 0.7.1. (https://archive.istio.io/v0.7/docs/concepts/what-is-istio/)
Download Istio files by running this

wget https://github.com/istio/istio/releases/download/0.7.1/istio-0.7.1-linux.tar.gz tar -xzf istio-0.7.1-linux.tar.gz rm istio-0.7.1-linux.tar.gz

Start installing Istio cd istio-0.7.1 export PATH=$PWD/bin:$PATH Apply Istio to kubernetes kubectl apply -f install/kubernetes/istio.yaml Just to check if istio elements are running, on both clusters run the following kubectl get pods -n istio-system -owide the output should be similar to this NAME READY STATUS RESTARTS AGE IP NODE istio-ca-77d7fb5cb9-4snrr 1/1 Running 0 3m 10.32.0.7 mcent1 istio-ingress-5c879987bf-j6n7z 1/1 Running 0 3m 10.32.0.5 mcent1 istio-mixer-d8b98df8f-4mnsv 3/3 Running 0 3m 10.32.0.4 mcent1 istio-pilot-74f45f5796-hj8nj 2/2 Running 0 3m 10.32.0.6 mcent1

Install Bookinfo.yaml from istio samples kubectl create -f <(istioctl kube-inject -f samples/bookinfo/kube/bookinfo.yaml) Check it the pods are running with the sidecars kubectl get pods -owide the output should be similar to this NAME READY STATUS RESTARTS AGE IP NODE details-v1-7bc7c59c8c-m7z8s 2/2 Running 0 1m 10.44.0.2 mcent2 productpage-v1-7c685d5cc9-spcpb 2/2 Running 0 1m 10.44.0.5 mcent2 ratings-v1-595bdd8c8c-2qphz 2/2 Running 0 1m 10.44.0.1 mcent2 reviews-v1-b6f5b6bc5-fnjmh 2/2 Running 0 1m 10.44.0.3 mcent2 reviews-v2-b7db5b9f-rbgr9 2/2 Running 0 1m 10.32.0.8 mcent1 reviews-v3-57c8bfbff9-jsznm 2/2 Running 0 1m 10.44.0.4 mcent2

Under the column READY it shows 2/2 which means this pods has two containers running (the micro-service and the sidecar)

To access the application that we just installed in the cluster, you need to get the port number that istio-ingress-5c879987bf-j6n7z is exposed on. Run the following kubectl get service -n istio-system the output should be similar to the this NAME TYPE CLUSTER-IP EXTERNAL-IP PORT(S) AGE istio-ingress LoadBalancer 10.106.59.14 <pending> 80:31515/TCP,443:30128/TCP 2d istio-mixer ClusterIP 10.109.110.193 <none> 9091/TCP,15004/TCP,9093/TCP,9094/TCP,9102/TCP,9125/UDP,42422/TCP 2d istio-pilot ClusterIP 10.101.116.201 <none> 15003/TCP,15005/TCP,15007/TCP,15010/TCP,8080/TCP,9093/TCP,443/TCP 2d As you can see right here that the istio-ingress serivce is exposed on 31515 for the enterprise cluster. Open a web browser and call the following IP address <The-Master's-IP>:31515/productpage to access the application. Do the same procedure to get the other cluster's port number. You will need the IP address and the port number to initiate the traces on each cluster.

To delete the bookinfo app, run kubectl delete -f samples/bookinfo/kube/bookinfo.yaml

To remove Istio from the cluster, run kubectl delete -f install/kubernetes/istio.yaml

4. Installing Zipkin
Run the following comand to install Zipkin

kubectl apply -f install/kubernetes/addons/zipkin.yaml need to get the port number that Zipkin is running on, run the follwoing ```

Now zipkin UI is accessible on the IP addresshttp://<master's-IP-Address>:```
