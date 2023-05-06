# Dockerization & Deployment of Nodejs App to Kubernetes
The process of dockerizing and deploying a Nodejs App to Kubernetes through CI/CD using GitHub Workflows can be broken down into the following steps:
* Fork App Repo
* Install & Config Minikube as Local Kubernetes
* Deploy Required DB for App
* Prepare Dockerfile for App
* Prepare Kubernetes Manifest for App
* Expose Local Machine for Remote SSH
* Prepare GitHub Workflows Action file (Dockerization & Deployment)
* Expose App to Public Internet

## Fork App Repo
Welcome to this repository, which is a fork of the original [flarie-todo](https://github.com/mistryflarie/flarie-todo) repository. By forking the repository, we have created a copy of the codebase that we can use for our own purposes.  
If you're interested in forking a repository yourself, GitHub provides documentation on how to do so in their [Fork a Repo](https://docs.github.com/en/get-started/quickstart/fork-a-repo) guide.  


## Install & Config Minikube as Local Kubernetes
To deploy our Nodejs App, we have installed and configured Minikube as our local Kubernetes cluster. As we don't have access to any cloud Kubernetes, this will allow us to deploy and test our app. To install Minikube locally on your machine, you can follow the official installation guide provided at [Start with Minikube](https://minikube.sigs.k8s.io/docs/start/). The guide outlines the steps to install the necessary dependencies and set up Minikube for your operating system.   
Before starting minikube, we need to ensure that our dockerized application can be deployed in our local Kubernetes and exposed using a NodePort service on port 34567. By default, minikube only exposes NodePort service ports in the range of 30000-32767, so we need to extend the port range by running the following command while starting the minikube locally:
```
minikube start --vm-driver=kvm2 --extra-config=apiserver.service-node-port-range=30000-35678
```
Here,
> --vm-driver, refers to the installed and configured virtualization technology driver to be used for the minikube cluster. For more info, follow [Minikube Drivers](https://minikube.sigs.k8s.io/docs/drivers/).    
> service-node-port-range, refers to the required NodePort service port range to be used for the minikube cluster.  


## Deploy Required DB for App
The forked Nodejs App is designed to work with either MySQL or SQLite DB. While SQLite is a server-less database and can be used in certain scenarios, for the purpose of providing a detailed and comprehensive technical demonstration, MySQL is preferred. To deploy MySQL to our local kubernetes, I have used the following kubernetes manifest file referred as [k8s-manifest/mysql-deployment.yml](https://github.com/junkcodes/flarie-todo/blob/main/k8s-manifest/mysql-deployment.yml).    
For configuring and deploying a MySQL database instance on Kubernetes, you can use the manifest file provided earlier. Otherwise for a more detailed guide, you can refer to the following link, [Run a Single-Instance Stateful MySQL Application](https://kubernetes.io/docs/tasks/run-application/run-single-instance-stateful-application/)

> The current version of MySQL 8 and the mysqljs npm package utilized in this application are currently facing authentication issues. This is because MySQL 8 is now using the caching_sha2_password pluggable authentication method as the default instead of the mysql_native_password, which is supported by the mysqljs package. To overcome this issue, get a shell into the running MySQL pod, login to MySQL as root and run the following command,
> ```
> ALTER USER 'username'@'%' IDENTIFIED WITH mysql_native_password BY 'password';
> FLUSH PRIVILEGES;
> ```


## Prepare Dockerfile for App
To dockerize the Nodejs App, I authored the Dockerfile referred as [dockerfile/Dockerfile](https://github.com/junkcodes/flarie-todo/blob/main/dockerfile/Dockerfile). The Dockerfile contains the following,
```
FROM node:lts
WORKDIR /app
COPY package.json .
COPY src .
RUN npm install
EXPOSE 3000
CMD ["node","index.js"]
```
For a more comprehensive guide to dockerize a Nodejs app, please refer to [Dockerizing a Node.js web app](https://nodejs.org/en/docs/guides/nodejs-docker-webapp).   


## Prepare Kubernetes Manifest for App
In order to deploy the Nodejs App in local Kubernetes, I have created a manifest file that you can find at [k8s-manifest/app-deployment.yml](https://github.com/junkcodes/flarie-todo/blob/main/k8s-manifest/app-deployment.yml).    
The manifest file defines both the Deployment and NodePort Service for the App with all the necessary configurations. By using this manifest file, we can quickly and easily deploy the application to Kubernetes cluster without having to manually configure all the necessary components.   


## Expose Local Machine for Remote SSH
To enable continuous deployment of our App using minikube on a local machine, we need to expose the machine for Remote SSH. This can be done by utilizing a tool known as Ngrok, which is a multi-platform tunnelling and reverse proxy software that creates secure tunnels from a public endpoint to a network service running locally. To install and set up Ngrok, you can follow the steps outlined in the following link: [Getting Started with ngrok](https://ngrok.com/docs/getting-started/).    
After properly setting up Ngrok, we can use its TCP Tunnels to expose our local machine for Remote SSH by the following command,
```
$ ngrok tcp 22
ngrok                                                                                         (Ctrl+C to quit)
                                                                                                              
Announcing ngrok-rs: The ngrok agent as a Rust crate: https://ngrok.com/rust                                  
                                                                                                              
Session Status                online                                                                          
Account                       Rezwan Mahmud Faisal (Plan: Free)                                               
Version                       3.2.2                                                                           
Region                        India (in)                                                                      
Latency                       117ms                                                                           
Web Interface                 http://127.0.0.1:4040                                                           
Forwarding                    tcp://0.tcp.in.ngrok.io:14593 -> localhost:22                                   
                                                                                                              
Connections                   ttl     opn     rt1     rt5     p50     p90                                     
                              6       0       0.00    0.00    0.02    5.22                                    
                                                                                   
```
Here,
> tcp://0.tcp.in.ngrok.io:14593 forwards all TCP requests to localhost:22   
> For SSH connection through TCP Tunnel tcp://0.tcp.in.ngrok.io:14593, provide HOSTNAME as 0.tcp.in.ngrok.io & PORT as 14593  


## Prepare GitHub Workflows Action file (Dockerization & Deployment)
Once all the necessary prerequisites and dependencies were in place, I created a CI/CD Workflow Action file called [.github/workflows/cicd.yml](https://github.com/junkcodes/flarie-todo/blob/main/.github/workflows/cicd.yml). This file automates the Dockerization and Deployment process to the local kubernetes, triggered by every push or merge request.

The action file contains two jobs, both of which have been implemented in two ways for better technical demonstration, as described below:
* build-image-(bangla-style/geeky-style)
* deploy-via-ssh-(bangla-style/geeky-style)

### build-image-(bangla-style/geeky-style)
The following job dockerizes the Nodejs App and pushes the resulting image to a public repository on Docker Hub referred as [junkcodes/flarie](https://hub.docker.com/r/junkcodes/flarie). Two different ways of implementing this job are described below.
#### build-image-bangla-style
The purpose of this Bangla Style implementation, is to utilize native Linux system techniques to complete the job, while minimizing the use of prebuilt GitHub Actions. The implementation is referred by the following,
```
  build-image-bangla-style:
    name: Build and Push App Image to Registry Bangla Style
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    - name: Login to Registry
      run: echo "${{ secrets.DOCKERPW }}" | docker login -u "junkcodes" --password-stdin
    - name: Build Image
      run: docker build -t junkcodes/flarie:bs -f dockerfile/Dockerfile .
    - name: Push Image
      run: docker push junkcodes/flarie:bs
```
#### build-image-geeky-style
The purpose of this Geeky Style implementation, is to maximize the use of prebuilt GitHub Actions. The implementation is referred by the following,
```
  build-image-geeky-style:
    name: Build and Push App Image to Registry Geeky Style
    runs-on: ubuntu-latest
    steps:
      - name: Set up QEMU
        uses: docker/setup-qemu-action@v2
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v2
      - name: Login to Docker Hub
        uses: docker/login-action@v2
        with:
          username: junkcodes
          password: ${{ secrets.DOCKERPW }}
      - name: Build and Push App Image
        uses: docker/build-push-action@v4
        with:
          file: dockerfile/Dockerfile
          push: true
          tags: junkcodes/flarie:gs
```


### deploy-via-ssh-(bangla-style/geeky-style)
The following job establishes SSH connection to our local machine to transfer the Kubernetes Manifest [k8s-manifest/app-deployment.yml](https://github.com/junkcodes/flarie-todo/blob/main/k8s-manifest/app-deployment.yml), and then executes the Manifest to properly deploy the dockerized App to the local kubernetes. Two different ways of implementing this job are described below.

#### deploy-via-ssh-bangla-style
The purpose of this Bangla Style implementation, is to utilize native Linux system techniques to complete the job, while minimizing the use of prebuilt GitHub Actions. The implementation is referred by the following,
```
  deploy-via-ssh-bangla-style:
    needs: build-image-bangla-style
    name: Deploy App to Kubernetes via SSH Bangla Style
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    - name: Configure SSH Client
      run: |
        mkdir -p ~/.ssh/
        echo "${{ secrets.SSH_PRIVATE_KEY }}" | base64 -d > ~/.ssh/vm.key
        chmod 600 ~/.ssh/vm.key
        cat >>~/.ssh/config <<END
        Host vm
          HostName 0.tcp.in.ngrok.io
          User groot
          IdentityFile ~/.ssh/vm.key
          StrictHostKeyChecking no
        END
    - name: Transfer K8s Manifest file for the App 
      run: scp -P 12904  k8s-manifest/app-deployment.yml vm:/tmp
    - name: Deploy App to the Kubernetes
      run: ssh -p 12904 vm "kubectl apply -f /tmp/app-deployment.yml -n flarie"
```

#### deploy-via-ssh-geeky-style
The purpose of this Geeky Style implementation, is to maximize the use of prebuilt GitHub Actions. The implementation is referred by the following,
```
  deploy-via-ssh-geeky-style:
    needs: build-image-geeky-style
    name: Deploy App to Kubernetes via SSH Geeky Style
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    - name: Prepare SSH Key
      run: echo "${{ secrets.SSH_PRIVATE_KEY }}" | base64 -d > vm.key
    - name: Transfer K8s Manifest file for the App 
      uses: appleboy/scp-action@v0.1.4
      with:
        host: 0.tcp.in.ngrok.io
        username: groot
        key_path: vm.key 
        port: 12904
        source: "k8s-manifest/app-deployment.yml"
        target: /tmp
        strip_components: 1
    - name: Deploy App to the Kubernetes
      uses: appleboy/ssh-action@v0.1.10
      with:
        host: 0.tcp.in.ngrok.io
        username: groot
        key_path: vm.key 
        port: 12904
        script: kubectl apply -f /tmp/app-deployment.yml -n flarie
```


## Expose App to Public Internet
After successfully creating and configuring the Github Workflows Action, the Nodejs App is ready to be dockerized and deployed automatically triggered by every push or merge request. The deployed App can be accessed locally using the address http://minikubeip:34567.   
However, wouldn't it be even more awesome if the app could be accessed on the public internet? To achieve this, we can utilize Ngrok to expose the Nodejs Web App using HTTP tunnels. Simply run the following command:
```
$ ngrok http http://192.168.39.42:34567
ngrok                                                                                         (Ctrl+C to quit)
                                                                                                              
Announcing ngrok-rs: The ngrok agent as a Rust crate: https://ngrok.com/rust                                  
                                                                                                              
Session Status                online                                                                          
Account                       Rezwan Mahmud Faisal (Plan: Free)                                               
Version                       3.2.2                                                                           
Region                        India (in)                                                                      
Latency                       -                                                                               
Web Interface                 http://127.0.0.1:4040                                                           
Forwarding                    https://5e06-202-74-48-126.ngrok-free.app -> http://192.168.39.42:34567         
                                                                                                              
Connections                   ttl     opn     rt1     rt5     p50     p90                                     
                              0       0       0.00    0.00    0.00    0.00                                                                                  
```
Here,
> For, ngrok http http://192.168.39.42:34567, 192.168.39.42 is the minikube ip. Run the command "minikube ip" to get minikube ip.   
> And https://5e06-202-74-48-126.ngrok-free.app is the public address for our deployed App running on http://192.168.39.42:34567
