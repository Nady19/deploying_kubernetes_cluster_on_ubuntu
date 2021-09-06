# Deploying kubernetes cluster

--> create 2 ec2 instances, one for master node(nady-kube-master) and another for worker node(nady-kube-worker)...N.b.: While doing that remember to choose t2 medium otherwise you cant initialize kubeadm.

-->now for both machines

`sudo apt update`

`sudo apt install docker.io -y`

->enabling docker on both instances:

`sudo systemctl start docker`

`sudo systemctl enable docker`

->downloading https and curl dependencies

`sudo apt install apt-transport-https curl`

->adding ubuntu repository to download kubernetes services

`curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add`

`sudo apt-add-repository "deb http://apt.kubernetes.io/ kubernetes-xenial main"`

->installing kubernetes modules:

`sudo apt install kubeadm kubelet kubectl kubernetes-cni`

->disabling swap memory:

`sudo swapoff -a`

-->To change the hostnames of both machines:-

->On master machine:

`sudo hostnamectl set-hostname kube-master`

->on worker machine:

`sudo hostnamectl set-hostname kube-worker1`

# On master machine

`sudo kubeadm init`

**warning:** kubelet is not running or healthy

go to `sudo vi /etc/docker/daemon.json`

input the following:

`{
"exec-opts": ["native.cgroupdriver=systemd"]
}`

restarting docker and kubelet:

`sudo systemctl daemon-reload`
 
`sudo systemctl restart docker`
 
`sudo systemctl restart kubelet`

resetting kubeadm

`sudo kubeadm reset`

`sudo kubeadm init`

Kubeconfig to access Kubectl command

`mkdir -p $HOME/.kube`

`sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config`

`sudo chown $(id -u):$(id -g) $HOME/.kube/config`

# On worker node: 

same.
(except the kubeconfig to access kubectl command part)

then from worker node:

`sudo kubeadm join 10.1.1.36:6443 --token 1zyxyl.m4wjgzqe8i3mrmfk --discovery-token-ca-cert-hash sha256:9186e7c4ef1d8bfaa3a605d245499580abfc1fa2cd32974ff33dbd7f5e744b2c`(6443 is the api server port)

(based on this token info from master machine)

# N.b.:
Never forget to put `sudo` before kubeadm...will give you a hard time if you forget...

# go to master node

`kubectl get pods`

`kubectl get pods -A`

to use a network fabric/POD network:

`kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"`

because if we hit `kubectl get nodes` we can see that the nodes aren't ready yet, but pretty soon after creating the pod network the nodes will be ready.

# deployment time

`vi deployment.yaml`

***paste the folowing :***

`apiVersion: apps/v1 #  for k8s versions before 1.9.0 use apps/v1beta2  and before 1.8.0 use extensions/v1beta1
kind: Deployment
metadata:
  name: frontend
spec:
  selector:
    matchLabels:
      app: guestbook
      tier: frontend
  replicas: 3
  template:
    metadata:
      labels:
        app: guestbook
        tier: frontend
    spec:
      containers:
      - name: php-redis
        image: nginx
        resources:
          requests:
            cpu: 100m
            memory: 100Mi
        env:
        - name: GET_HOSTS_FROM
          value: dns
          # If your cluster config does not include a dns service, then to
          # instead access environment variables to find service host
          # info, comment out the 'value: dns' line above, and uncomment the
          # line below:
          # value: env
        ports:
        - containerPort: 80`

N.b.: press `qa` twice to insert, `esc` to end/cancel editing, `:wq` to save and exit        


`kubectl apply -f deployment.yaml`

to check and confirm the deploymment of pods :

`kubectl get pods`

we can delete pods by `kubectl delete pods *name of a pod*`

but if you `kubectl get pods` there will be another pod just respawned because of the command `replicas: 3` in the deployment file.

# Now

`vi service.yaml`

***paste the folowing :***

`apiVersion: v1
kind: Service
metadata:
  name: frontend
  labels:
    app: guestbook
    tier: frontend
spec:
  type: NodePort
  ports:
  - port: 80
  selector:
    app: guestbook
    tier: frontend`


`kubectl apply -f service.yaml`

now we can check the services and endpoints :

`kubectl get svc`

`kubectl get ep`

now lets go to the public ip of worker node from browser with the port of frontend on the master node which is 32481

from browser `http://13.232.69.8:32481/`...you can see the nginx page !!!!

To simply put it ; deployment(an octopus) keeps the pods alive and and the traffic/request is reached to the pods through service.
