# Directeam DevOps interview - ArgoCD
![Build Status](https://github.com/directeam-io/DevOps-Interview-ArgoCD/actions/workflows/ci.yml/badge.svg)



I would like to mention that I do not have experience hands-on with k8s, same for ArgoCD.
and this was the first time I have actually worked with it.

First I have installed VMware Workstation.
I have then downloaded Ubuntu20 ISO and configured it into the workstation, settings it up completely from 0, User is "Meidan"
I have looked up for minikube documentation to learn how to install it.
Installed Docker on my machine = added the repo as in docs, yum install docker-ce (etc), then started the service with systemctl start docker
checked, docker's running
sudo usermod -aG docker Meidan
installed minikube on my machine following the docs:
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube_latest_amd64.deb
sudo dpkg -i minikube_latest_amd64.deb
then ran minikube start. it downloaded some deps (kubectl etc)
minikube kubectl is up :)
cd'd to Desktop, cloned git repo there (creating folder w repo)
git clone https://github.com/meidal11/DevOps-Interview-ArgoCD
set up an SSH key for the machine:
ssh-keygen -p -f ~/.ssh/id_rsa
created SSH as normal
eval "$(ssh-agent -s)"
agent running

added SSH key on git website and pushed the cloned repo - worked.
enabled github actions through github, went to forked repo > settings > actions and enabled it there.
followed agroCD's docs to install:
minikube kubectl -- apply --namespace agrocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'
just incase did port forwarding: kubectl port-forward svc/argocd-server -n argocd 8080:443
couldn't access anything on K8s with the error "	"forbidden: User \"system:anonymous\" cannot get path \"/\"""
found a workaround: kubectl create clusterrolebinding cluster-system-anonymous --clusterrole=cluster-admin --user=system:anonymous (gives everyone permissions, but shouldn't matter in this scenario.)
another possibility: (minikube start?)"--anonymous-auth=false" 
got ArgoCD's password with: kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
pass: nj8ATrc4ptmru04y <<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<< (for myself)
installed argoCD CLI with:
curl -sSL -o /usr/local/bin/argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
gave it chown +x /usr/loca/bin/argocd
connected with argocd login 127.0.0.1:8080
then not secured
user: admin
pass: nj8ATrc4ptmru04y
logged in!
after some playing around, I changed the service from ClusterIP to NodePort:
minikube kubectl -- get all -n argocd
got the service
minikube kubectl -- -n argocd edit svc argocd-server
it opened a vi, scrolled down and changed the ClusterIP to NodePort, then saved it
checked again with the get all command, it did change to NodePort
was able to login to ArgoCD with [my ipv4 ip]:[port that the service was attached to]
then i went into the ArgoCD UI and connected the forked repository https://github.com/meidal11/DevOps-Interview-ArgoCD

I then built the dockerfile by CDing into the folder and "docker build . -t argodocker:1"
connected to docker with docker login etc etc

connected git repo through ArgoCD's UI, username, password etc
I then manifested everything that's necessary for ArgoCD, therefore:

touch Deployments.yaml, app.yaml etc etc

Deployment.yaml:

apiVersion: apps/v1
kind: Deployment
metadata:
  name: argodep
spec:
  selector:
    matchLabels:
      app: default
  replicas: 3
  template:
    metadata:
      labels:
        app: default
    spec:
      containers:
      - image: meidal11/argodocker:3
        name: argoapp
        ports:
        - containerPort: 8080

Application.yaml:

apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
        name: argoapp
        namespace: argocd
spec:
        project: default
        source:
                repoURL: https://github.com/meidal11/DevOps-Interview-ArgoCD.git
                targetRevision: HEAD
                path: .
        destination:
                server: https://kubernetes.default.svc
                namespace: default
        syncPolicy:
                syncOptions:
                        - CreateNamespace=true
                automated:
                        selfHeal: true
                        prune: true


I then deployed the app:
kubectl apply -f application.yaml
it all started properly :)
I then changed the README.md and pushed into git - after around 2min of monitoring the ArgoCD went out of sync and resynced
I pushed it once again and changed the replicas on deployment.yaml to 2 - it updated with 2 replicas
edited back to 3 replicas - again took like 2-3 min which is kind of long imo, but it works :)
conclusion: every change that has been pushed to the Git repo is automatically being updated into ArgoCD within my Minikube session.

Creating NodePort Service:
touch svc.yaml

apiVersion: v1
kind: Service
metadata:
  name: argosvc
spec:
  type: NodePort
  selector:
    app: default
  ports:
    - protocol: TCP
      port: 8080


added this service to git to run automatically. (git add, commit, push etc)
synced ArgoCD, service created (automatically)
checked with 'kubectl get all' - saw it there, called "svc/argosvc'
then 'minikube service argosvc' - showed basic info and opened the webpage with port 30764 - error 'no user header found'
curl'd it, got a response again of no header - curl 192.168.49.2:30764
it kept on saying 'no header found' and after some investigating in the main.go file i realized I need to add an Auth to the CURL, therefore:
curl -X POST -H "X-Auth-Request-User: Meidan" 192.168.49.2:30764 (30764 was the random port)
got an answer: "Hello meidan"

Edit some of the source code:
vi main.go
changed the "hello %s" to "Everybody be quiet, %s just joined!"
realized i need to change the deployment.yaml, and so I did - image: meidal11/argodocker:2 (you asked not to use latest)
docker build . -t meidal11/argocd:2
docker push meidal11/argocd:2
saved, git status - changed, git add main.go, git commit, pushed.
I wanted it to load automatically, so i gave it 3min.
It update automatically, but to make things faster:
kubectl delete deployment/argodep
curl -X POST -H "X-Auth-Request-User: Meidan" 192.168.49.2:30764
got response: Everybody be quiet, Meidan just joined!
Worked, mission's over



Some problems I had:
Image kept on pulling back. Apparently it didn't pull from my local repository (thought by common sense that it would work), so i pushed the docker image and pointed the deployment to look over to meidal11/argo instead of locally.
I couldn't access the web hello-world because I couldn't expose port 8080 as it was already used locally. after a lot of headache I realized ArgoCD's UI was using it... changed the ArgoCD's port-forward to 8090
I still couldn't access the web hello-world. I accidently pushed port 80 and not 8080.
I needed to 'kubectl delete svc' a serice every time after i synced it on argocd for faster sync
