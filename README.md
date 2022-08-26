In an era of Kubernetes and container-based applications, it is very important to know Go Language as well. Go Lang is one of the popular (might be most popular) languages for developing Microservices or Enterprise Applications in Kubernetes or OpenShift. In this article, we will learn how to deploy the Go application in Kubernetes.

I have used the following technologies for developing/deploying Go Application:

1. minikube v1.17.1 on Fedora 33

2. Docker/Podman

3. Visual Studio Code 1.51.1

Step 1
Create a file firstGoInKubernetes.go with the following content. This is a simple web server application listening on port 9090.

Go

1 package main
2
3 import (
4    "fmt"
5    "log"
6    "net/http"
7)
8
9 func main() {
10    http.HandleFunc("/", handlerFunc)
11    log.Fatal(http.ListenAndServe("0.0.0.0:9090", nil))
12}
13
14 func handlerFunc(w http.ResponseWriter, r *http.Request) {
15    log.Printf("Ping from %s", r.RemoteAddr)
16    fmt.Fprintln(w, "Hello, WebServer is listening on  9090")
17
18}
19

Step 2
Before creating a podman/docker image, let us first test this application is ok or not:

Shell
1 $ go run firstGoInKubernetes.go
2 # in different terminal
3 $ curl http://0.0.0.0:9090
4 Hello, WebServer is listening on  9090

Step 3
As we have tested the application and it provides the desired result, we will now go build to compile the package and dependencies. GOOS=linux and GOARCH=amd64 because I am running this in Fedora 33. CGO_ENABLE=0 creates a standalone binary which is ideal for docker images.

Shell
1 $ CGO_ENABLE=0 GOOS=linux GOARCH=amd64 go build -o firstGoImage
2 $ ls -ltr
3 -rwxrwxr-x. 1 tester tester 		6445426 	Feb 21 17:58 		firstGoImage
4 -rw-rw-r--. 1 tester tester      	60 				Feb 21 18:03 		Dockerfile
5 -rw-r--r--. 1 tester tester     		308 			Feb 21 18:35 		firstGoInKubernetes.go

Step 4
In the same folder location of this firstGoInKubernetes.go, we can create a Docker file Dockerfile:

Dockerfile
1 FROM alpine:latest
2 WORKDIR deploy
3 COPY firstGoImage /deploy/
4 EXPOSE 9090
5 CMD ["/deploy/firstGoImage"]

Step 5
Start minikube. I am using the profile Go-POC. Also, point docker context to minikube's docker registry:

Shell

1 #start minikube
2 $ minikube start -p Go-POC
3 # point docker registry to minikube's with Go-POC profile
4 $ eval $(minikube -p Go-POC docker-env)
5 # check if docker registry is correctly set
6 $ minikube -p Go-POC ip
7 192.168.39.21
8 $ docker context ls
9 NAME                DESCRIPTION                               						DOCKER ENDPOINT            	KUBERNETES ENDPOINT                    ORCHESTRATOR
10 default *           Current DOCKER_HOST based configuration   tcp://192.168.39.21:2376   		https://192.168.39.21:8443 (default)   	swarm
11
12 $ 
13




Step 6
Build docker image:

Shell

1 $ docker build -t first-go-image:v1.0 .
2
3 $ docker images
4 REPOSITORY                                					TAG                 IMAGE ID            	  	CREATED              			SIZE
5 first-go-image                            							v1.0      			4491e85f623b        	About a minute ago   		12.1MB
6 alpine                                    								latest 				28f6e2705743        	3 days ago           				5.61MB
7 k8s.gcr.io/kube-proxy                     					v1.20.2           43154ddb57a8        	5 weeks ago          			118MB
8 k8s.gcr.io/kube-controller-manager        			v1.20.2          	a27166429d98        	5 weeks ago          			116MB
9 k8s.gcr.io/kube-apiserver                 				v1.20.2       	a8c2fdb8bf76        	5 weeks ago          			122MB
10 k8s.gcr.io/kube-scheduler                 				v1.20.2          	ed2c44fbdd78        	5 weeks ago          			46.4MB
11 kubernetesui/dashboard                    				v2.1.0            	9a07b5b4bfac        	2 months ago         			226MB
12 gcr.io/k8s-minikube/storage-provisioner   	v4                  	85069258b98a        	2 months ago         			29.7MB
13 k8s.gcr.io/etcd                           						3.4.13-0         	0369cf4303ff        		5 months ago         			253MB
14 k8s.gcr.io/coredns                        					1.7.0              	bfe3a36ebd25        	8 months ago         			45.2MB
15 kubernetesui/metrics-scraper              			v1.0.4            	86262685d9ab        	11 months ago        			36.9MB
16 k8s.gcr.io/pause                          					3.2                 	80d28bedfe5d        	12 months ago        			683kB
17 $ 
18

Step 7
Create Kubernetes Deployment and expose a NodePort service for this deployment:

Shell

1 $ kubectl create deployment gofirstimage --image=first-go-image:v1.0 --replicas=1
2 deployment.apps/gofirstimage created
3
4 $ kubectl expose deployment gofirstimage --name=go-service --port=9090 --target-port=9090 --type=NodePort
5 service/go-service exposed
6
7 $ kubectl get all
8 NAME                               					READY   STATUS    	RESTARTS   AGE
9 pod/gofirstimage-fd8879bdd-zkdbj   1/1     		Running   					0          20m
10
11 NAME                 			TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)          		AGE
12 service/go-service   NodePort   10.99.180.230   			 <none>        9090:31614/TCP     12m
13 
14 NAME                           				READY   UP-TO-DATE   AVAILABLE   AGE
15 deployment.apps/gofirstimage   1/1     					1            			1             20m
16
17 NAME                                     						DESIRED   CURRENT   READY   AGE
18 replicaset.apps/gofirstimage-fd8879bdd   		1         			1         			1         20m
19 $ 

Step 8
Access Service:

Shell

1 # check minikube IP. This is IP of node where NodePort is exposed.
2
3 $ minikube ip -p Go-POC
4 192.168.39.21
5
6 $ kubectl get svc
7 NAME         TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
8 go-service   NodePort   10.99.180.230   <none>        9090:31614/TCP   17m
9
10 $ curl http://192.168.39.21:31614
11 Hello, WebServer is listening on  9090
12
13 $ kubectl logs -f gofirstimage-fd8879bdd-zkdbj
14 2021/02/22 09:08:24 Ping from 172.17.0.1:15745

Step 9
Further Troubleshooting: Check the content of the docker image to find if Go binary is available:

Shell

1 $ eval $(minikube -p Go-POC docker-env)
2
3 $ docker ps |grep firstGo
4 CONTAINER ID        IMAGE                  COMMAND                  CREATED             STATUS              PORTS               NAMES
5
6 69c0567525f4        79950e6f3c45           "/deploy/firstGoImage"   31 minutes ago      Up 31 minutes                           k8s_first-go-image_gofirstimage-fd8879bdd-zkdbj_goproject_c6764971-fbaa-4f7c-bbc9-7c41e156d968_0
7
8 $ docker export -o image.tar 69c0567525f4
9
10 $ tar xvf image.tar
11
12 $ cd deploy/
13
14 $ ls -ltr
15 total 6248
16 -rwxrwxr-x. 1 tester tester 6393972 Feb 21 22:50 firstGoImage


Wrap Up
That's it, guys. I believe these steps would help you to have a better understanding of developing + deploying + troubleshooting the Go Lang Application in Kubernetes or OpenShift.