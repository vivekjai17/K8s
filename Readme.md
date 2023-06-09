How to set up the k8s cluster and deploy applications in the pods using  AWS - EKS & ECR.


Step: 1 - 

Create an EC2 Instance to control the EKS Worker nodes

Create an ec2 instance to manage the Worker nodes, create clusters, create services, and deploy pods.

Step: 2 - 

Install docker in the ec2 Instance created.

#sudo apt install docker.io

Step:3 

Install  AWS-CLI(Command line interface) and configure the AWS account using AWS Access Key ID and AWS Secret Access Key using the below-mentioned commands.

# sudo curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
# unzip awscliv2.zip
# sudo apt install unzip
# unzip awscliv2.zip
# sudo ./aws/install
# /usr/local/bin/aws --version
# sudo aws configure ( user you Access Key ID and AWS Secret Access Key )


Step:4


Install kubectl using the below command

# curl -LO https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl
 # chmod +x ./kubectl
 # sudo mv ./kubectl /usr/local/bin/kubectl

Step:5 

Create a cluster in the AWS console 

Login to your AWS console and Goto - Search and type “EKS”
Now click on the “Elastic Kubernetes Service” menu



Now click “Add cluster” and then click “create” and Enter the details to Configure cluster



=> Enter Cluster Name - LF-Cluster
=> Select Kubernetes version - 1.27
=> Cluster service role - dev-cluster-role
     (we will be providing access to other AWS services for this cluster)
=> Click Next


=> Now enter networking details - Select VPC, Subnets, Security-groups
=> Choose cluster IP address family - IPv4

=> Cluster endpoint access - Public

=> Click “Next” and in this page(Configure logging) keep it as default settings and click “Next”
=> In Select add-ons page click “Next” with default selections.
=> In Configure selected add-ons settings page click “Next”
=> Now review the configurations and click “Create”

Now the cluster will be created in 10-15 minutes.



Once the cluster status becomes “Active” we can able to create the node.




Step:6 

Now go to the EC2 Instance that we created to update the cluster that we created in the AWS console.

# aws eks --region ap-south-1 update-kubeconfig --name LF-cluster

Now check the updated cluster list in your instance 

# aws eks list-clusters





Once the created cluster is listed we can create the worker nodes using kubectl command.










Step:7 


Create the worker nodes using kubectl. This will help to create the worker node in a private VPC without public IP


# aws eks create-nodegroup --cluster-name <cluster-name> --nodegroup-name <nodegroup-name> --scaling-config minSize=1,maxSize=10,desiredSize=3 --instance-types t3.medium --subnets subnet-12345678,subnet-87654321 --ami-type AL2_x86_64 --node-role-arn arn:aws:iam::<account-id>:role/<role-name> --remote-access ec2SshKey=<key-pair-name>,sourceSecurityGroups=<security-group-id>

=> By entering the desired values in the above-mentioned command we can able to create the worke-node-group and nodes with the specs mentioned.

# sudo aws eks create-nodegroup --cluster-name LF-cluster --nodegroup-name LF-node-group --scaling-config minSize=2,maxSize=2,desiredSize=2 --instance-types t3.xlarge --subnets subnet-076b36e039361a488 --ami-type AL2_x86_64 --node-role arn:aws:iam::921015586233:role/dev-cluster-worker-node-role --disk-size 80 --remote-access ec2SshKey=prod-blog,sourceSecurityGroups=sg-0577bfb273ed32b31


Now check the aws console in EKS > Cluster > LF-Cluster > Compute . There will be two instances(nodes)  available in ready status.

Use this command to check the details of the node-group and nodes created in cli.

# sudo aws eks describe-nodegroup --cluster-name LF-cluster --nodegroup-name LF-node-group


Step:8 

Install ingress-nginx controller service to expose the pods to the public

# sudo git clone https://github.com/kubernetes/ingress-nginx.git
# cd ingress-nginx/deploy/static/provider/aws
# kubectl apply -f deploy.yaml

Now the ingress-nginx controller is installed. 

Check the service using kubectl command







# kubectl get pods -n  ingress-nginx -o wide



# kubectl get service ingress-nginx-controller --namespace=ingress-nginx



We get the external-IP which is a load balancer endpoint created on AWS.

Now if we browse the external-IP we will be able to see the nginx 404 error page. This means that the load balancer configuration is working and it has no applications to serve.

Next we can set up the application from AWS-ECR images.



Step:9

Now we have to set up the applications and the services using the YAML files.

Create a yaml file using vi command

# vi dockerization.yaml


apiVersion: apps/v1
kind: Deployment
metadata:
  name: advance-website
  # namespace: dockerization
  labels:
    app: advance-website
spec:
  selector:
    matchLabels:
      app: advance-website
  replicas: 2
  template:
    metadata:
      labels:
        app: advance-website
    spec:
      serviceAccountName: ingress-nginx
      containers:
      - name: advance-website
        image: 921015586233.dkr.ecr.ap-south-1.amazonaws.com/dockerization:advance-website
        ports:
            - containerPort: 3000
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: finance-frontend
  #namespace: dockerization
  labels:
    app: finance-frontend
spec:
  selector:
    matchLabels:
      app: finance-frontend
  replicas: 2
  template:
    metadata:
      labels:
        app: finance-frontend
    spec:
      serviceAccountName: ingress-nginx
      containers:
      - name: finance-frontend
        image: 921015586233.dkr.ecr.ap-south-1.amazonaws.com/dockerization:finance-frontend
        ports:
            - containerPort: 3002
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: leap-portal
  #namespace: dockerization
  labels:
    app: leap-portal
spec:
  selector:
    matchLabels:
      app: leap-portal
  replicas: 2
  template:
    metadata:
      labels:
        app: leap-portal
    spec:
      serviceAccountName: ingress-nginx
      containers:
      - name: leap-portal
        image: 921015586233.dkr.ecr.ap-south-1.amazonaws.com/dockerization:leap-portal
        ports:
            - containerPort: 3001
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: leap-backend
  #namespace: dockerization
  labels:
    app: leap-backend
spec:
  selector:
    matchLabels:
      app: leap-backend
  replicas: 2
  template:
    metadata:
      labels:
        app: leap-backend
    spec:
      serviceAccountName: ingress-nginx
      containers:
      - name: leap-backend
        image: 921015586233.dkr.ecr.ap-south-1.amazonaws.com/dockerization:leap-backend
        ports:
            - containerPort: 8080
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: advance-backend
  #namespace: dockerization
  labels:
    app: advance-backend
spec:
  selector:
    matchLabels:
      app: advance-backend
  replicas: 2
  template:
    metadata:
      labels:
        app: advance-backend
    spec:
      serviceAccountName: ingress-nginx
      containers:
      - name: advance-backend
        image: 921015586233.dkr.ecr.ap-south-1.amazonaws.com/dockerization:advance-backend
        ports:
            - containerPort: 8000
---
apiVersion: v1
kind: Service
metadata:
  name: advance-website
spec:
  type: ClusterIP
  ports:
  - port: 80
    targetPort: 3000
  selector:
    app: advance-website
  ports:
  - port: 443
    targetPort: 3000
  selector:
    app: advance-website
---
apiVersion: v1
kind: Service
metadata:
  name: finance-frontend
spec:
  type: ClusterIP
  ports:
  - port: 80
    targetPort: 3002
  selector:
    app: finance-frontend
  ports:
  - port: 443
    targetPort: 3002
  selector:
    app: finance-frontend
---
apiVersion: v1
kind: Service
metadata:
  name: leap-portal
spec:
  type: ClusterIP
  ports:
  - port: 80
    targetPort: 3001
  selector:
    app: leap-portal
  ports:
  - port: 443
    targetPort: 3001
  selector:
    app: leap-portal
---
apiVersion: v1
kind: Service
metadata:
  name: leap-backend
spec:
  type: ClusterIP
  ports:
  - port: 80
    targetPort: 8080
  selector:
    app: leap-backend
  ports:
  - port: 443
    targetPort: 8080
  selector:
    app: leap-backend
---
apiVersion: v1
kind: Service
metadata:
  name: advance-backend
spec:
  type: ClusterIP
  ports:
  - port: 80
    targetPort: 8000
  selector:
    app: advance-backend
  ports:
  - port: 443
    targetPort: 8000
  selector:
    app: advance-backend
---


Save the file and next we have to apply the file.

Now apply the file using kubectl command to deploy the pods  with the namespace ingress-nginx

# kubectl apply -f dockerization.yaml --namespace ingress-nginx

Once we run this command the pods get deployed and we can check it using the below command

# kubectl get pods -n ingress-nginx -o wide





We will be able to see the pods that are in running state with the names that we provided.




Step:10

Set up the Ingress to route the traffic between the pods that we deployed.


Create the ingress.yaml file and here we will be using domain-based routing using the service name.

# vi ingress.yaml

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: advance-website
spec:
  ingressClassName: nginx
  rules:
  - host: eks-advancewebsite.leapfinance.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: advance-website
            port:
              number: 443
      - path: /refinance
        pathType: Prefix
        backend:
          service:
            name: finance-frontend
            port:
              number: 443
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: financefrontend
spec:
  ingressClassName: nginx
  rules:
  - host: eks-financefrontend.leapfinance.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: financefrontend
            port:
              number: 443
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: leap-portal
spec:
  ingressClassName: nginx
  rules:
  - host: eks-leaportal.leapfinance.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: leap-portal
            port:
              number: 443
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: leap-backend
spec:
  ingressClassName: nginx
  rules:
  - host: eks-leapbackend.leapfinance.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: leap-backend
            port:
              number: 443
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: advance-backend
spec:
  ingressClassName: nginx
  rules:
  - host: eks-advancebackend.leapfinance.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: leap-backend
            port:
              number: 443


Save the file and exit.

Now we have to apply the ingress.yaml file to create the routing.
# kubectl apply -f ingress.yaml --namespace ingress-nginx

Check the services deployed using the kubectl command

# kubectl get svc -n ingress-nginx






Step:11

Now create the domain individually for all the applications that we mapped with the ingress.yaml file. in AWS route53 and map the loadbalancer Extrnal-IP that we see in the above step.


Go to AWS console route53 and select the domain then “create record” enter the domain name and the value as loadbalancer Extrnal-IP 
Then select the Record type as CNAME and click create records.



Now hit the domain that you have created in route53 we are able to see the application page.

Step:12

How to add another IAM user to access a cluster

Create an aws-auth.yaml file 


# vi aws-auth.yaml

apiVersion: v1
kind: ConfigMap
metadata:
  name: aws-auth
  namespace: kube-system
data:
  mapRoles: |
    - rolearn: arn:aws:iam::921015586233:role/dev-cluster-worker-node-role
      username: system:node:{{EC2PrivateDNSName}}
      groups:
        - system:bootstrappers
        - system:nodes
  mapUsers: |
    - userarn: <IAM user ARN>
      username: <username>
      groups:
        - system:masters

# sudo kubectl apply -f aws-auth.yaml




Step:13

Install and configure cert-manager and apply SSL.

Run this command to install cert manager

# kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.7.1/cert-manager.yaml


To verify the installation just run this command

# kubectl get pods --namespace cert-manager



we need to create an Issuer, which specifies the certificate authority from which signed x509 certificates can be obtained. In this guide, we’ll use the Let’s Encrypt certificate authority, which provides free TLS certificates 


Create a file prod_issuer.yaml

# vi prod_issuer.yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
  namespace: cert-manager
spec:
  acme:
    # The ACME server URL
    server: https://acme-v02.api.letsencrypt.org/directory
    # Email address used for ACME registration
    email: support@pheonixsolutions.com
    # Name of a secret used to store the ACME account private key
    privateKeySecretRef:
      name: letsencrypt-prod
    # Enable the HTTP-01 challenge provider
    solvers:
    - http01:
        ingress:
          class: nginx

Now save the file and exit.



Now roll out this issuer using kubectl command

# kubectl create -f prod_issuer.yaml

Now check the created issuer using this command

# kubectl get clusterissuer



To Issue SSL for the domain we have to edit the ingress.yaml file,


# vi ingress.yaml

 apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: advance-website
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-staging"
    kubernetes.io/ingress.class: "nginx"
spec:
  tls:
  - hosts:
    - eks-advancewebsite.leapfinance.com
    secretName: eks-advancewebsite.leapfinance.com-tls
  ingressClassName: nginx
  rules:
  - host: eks-advancewebsite.leapfinance.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: advance-website
            port:
              number: 443
      - path: /refinance
        pathType: Prefix
        backend:
          service:
            name: finance-frontend
            port:
              number: 443
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: financefrontend
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-staging"
    kubernetes.io/ingress.class: "nginx"
spec:
  tls:
  - hosts:
    - eks-financefrontend.leapfinance.com
    secretName: eks-financefrontend.leapfinance.com-tls
  ingressClassName: nginx
  rules:
  - host: eks-financefrontend.leapfinance.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: financefrontend
            port:
              number: 443
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: leap-portal
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-staging"
    kubernetes.io/ingress.class: "nginx"
spec:
  tls:
  - hosts:
    - eks-leaportal.leapfinance.com
    secretName: eks-leaportal.leapfinance.com-tls
  ingressClassName: nginx
  rules:
  - host: eks-leaportal.leapfinance.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: leap-portal
            port:
              number: 443
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: leap-backend
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-staging"
    kubernetes.io/ingress.class: "nginx"
spec:
  tls:
  - hosts:
    - eks-leapbackend.leapfinance.com
    secretName: eks-leapbackend.leapfinance.com-tls
  ingressClassName: nginx
  rules:
  - host: eks-leapbackend.leapfinance.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: leap-backend
            port:
              number: 443
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: advance-backend
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-staging"
    kubernetes.io/ingress.class: "nginx"
spec:
  tls:
  - hosts:
    - eks-advancebackend.leapfinance.com
    secretName: eks-advancebackend.leapfinance.com-tls
  ingressClassName: nginx
  rules:
  - host: eks-advancebackend.leapfinance.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: advance-backend
            port:
              number: 443


Now save the file and apply it.

# kubectl apply -f ingress.yaml -n ingress-nginx


Step:14


Now create the <domain>-cert.yaml file to get the certificates for each domain.

# vi advancefe_cert.yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: eks-advancewebsite.leapfinance.com
  namespace: cert-manager
spec:
  secretName: eks-advancewebsite.leapfinance.com-tls
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  commonName: eks-advancewebsite.leapfinance.com
  dnsNames:
  - eks-advancewebsite.leapfinance.com

Now save this file and apply it

# kubectl apply -f advancefe_cert.yaml

Same way create new files for all the domains and apple it. 

Now we can able to access the website using letsencrypt ssl applied.
