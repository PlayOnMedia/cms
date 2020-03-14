# AWS / KOPS / HELM

* https://kops.sigs.k8s.io/getting_started/aws/

## Requirements

* **Kubectl:** https://kubernetes.io/docs/tasks/tools/install-kubectl/
* **KOPS:** https://github.com/kubernetes/kops
* **Helm:** https://helm.sh/docs/using_helm/
* **JQ** sudo apt-get install jq
* **AWS CLI:** https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-install.html
* **AWS Access Creds:** https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-configure.html
* **AWS S3 Bucket:** https://docs.aws.amazon.com/cli/latest/reference/s3api/create-bucket.html

AWS CLI
```
sudo apt update
sudo apt install -y awscli
aws config
```
KUBECTL
```
sudo snap install kubectl --classic
or
curl -LO https://storage.googleapis.com/kubernetes-release/release/`curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt`/bin/linux/amd64/kubectl
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin/kubectl
```
KOPS
```
wget https://github.com/kubernetes/kops/releases/download/v1.16.0/kops-linux-amd64
chmod +x kops-linux-amd64
mv ./kops-linux-amd64 /usr/local/bin/kops
```
HELM
```
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```

Setup AWS credentials locally (~/.aws/credentials) to create the s3 bucket/run KOPS commands
```
[default]
aws_access_key_id = XXXXXXXXXXXXXXXX
aws_secret_access_key = XXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
```

Update your environment file (First copy .env.sample to .env and after filling it out run source .env)

Create a bucket to store the KOPS state.
```
aws s3api create-bucket --bucket $S3_BUCKET --region us-east-1
aws s3api put-bucket-versioning --bucket $S3_BUCKET --versioning-configuration Status=Enabled
export KOPS_STATE_STORE=s3://kops-state-playonmedia
```

KOPS CLUSTER ACCOUNT
```
aws iam create-group --group-name kops
aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonEC2FullAccess --group-name kops
aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonRoute53FullAccess --group-name kops
aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess --group-name kops
aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/IAMFullAccess --group-name kops
aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonVPCFullAccess --group-name kops
aws iam create-user --user-name kops
aws iam add-user-to-group --user-name kops --group-name kops
aws iam create-access-key --user-name kops
```

## Create Cluster

### With Subdomain
https://github.com/kubernetes/kops/blob/master/docs/getting_started/aws.md

```
ID=$(uuidgen) && aws route53 create-hosted-zone --name cluster.$DOMAIN_NAME --caller-reference $ID | jq .DelegationSet.NameServers
aws route53 list-hosted-zones | jq '.HostedZones[] | select(.Name=="$DOMAIN_NAME.") | .Id'
touch subdomain.json
export HOSTED_ZONE_ID=
aws route53 change-resource-record-sets --hosted-zone-id $HOSTED_ZONE_ID --change-batch file://subdomain.json
dig ns cluster.$DOMAIN_NAME
```


Run locally or add add to `vi ~/.bash_profile` or equiv

This allows us to avoid using `--state=s3://kops-state-playonmedia` with `kops` `create`, `update`, `validate`, and `delete` commands.

```
export KOPS_STATE_STORE=s3://kops-state-playonmedia
```

```
kops create cluster --name=cluster.$DOMAIN_NAME --zones=us-east-1a --node-count=2 --node-size=t2.micro --master-size=t2.micro --state=s3://$S3_BUCKET --yes
kops validate cluster
```

### Without Subdomain
```
kops create cluster --name=playonmedia.com --state=s3://kops-state-playonmedia --zones=us-east-1a --node-count=2 --node-size=t2.micro --master-size=t2.micro
kops update cluster playonmedia.com --yes --state=s3://kops-state-playonmedia 
kops validate cluster --state=s3://kops-state-playonmedia
``` 

## Setup Helm & Installing a Chart
### Installing Tiller on the cluster
```
<!-- kubectl create serviceaccount --namespace kube-system tiller
kubectl create clusterrolebinding tiller-cluster-rule --clusterrole=cluster-admin --serviceaccount=kube-system:tiller
helm init --service-account tiller -->

helm init
helm repo add stable https://kubernetes-charts.storage.googleapis.com/

```

### Installing a WordPress example chart
```
helm install stable/wordpress --generate-name wordpress
helm list
helm del --purge wordpress
```

## Deploy/View Resources (Example)
```
kubectl get nodes 
kubectl run hello-minikube --image=k8s.gcr.io/echoserver:1.4 --port=80 
kubectl expose deployment hello-minikube --type=NodePort 
kubectl get services
``` 

Modify the inbound security group for the nodes to allow accesss from your IP address or re-deploy the top resource as a `--type=Loabalancer` to have a AWS ALB as the endpoint to the internet. You can find the ipv4 address from `kubectl` or the `ec2 instance` in AWS. Make sure you access this via the exposed port from the deployment above.

## Nginx Example
```
kubectl run my-nginx-app --image nginx:latest
kubectl expose deployment my-nginx-app --port=80 --type=LoadBalancer
kubectl describe svc my-nginx-app
```

## Scaling nodes

Update listed nodes from minimum 2 to 5 and a max of 8

```
kops edit ig --name= nodes --state=s3://kops-state-playonmedia
kops update cluster --state=s3://kops-state-playonmedia
kops update cluster --state=s3://kops-state-playonmedia --yes
kops rolling-update cluster --state=s3://kops-state-playonmedia --yes
kops get instancegroups --state=s3://kops-state-playonmedia
```

## View Dashboard (Web UI)

```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0-beta4/aio/deploy/recommended.yaml
kubectl proxy
http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/
```

## Delete Cluster

* `kops delete cluster cluster.playonmedia.com --state=s3://kops-state-playonmedia --yes`

## MISC

* https://landscape.cncf.io/format=card-mode
* https://hub.helm.sh/

## TO DO
* Create all of these resources with Terraform
* Prometheus
* Fluentd

## Gitlab Setup (WIP)

* `.gitlab-ci.yml` -> `AWS_CREDENTIALS & KOPS_KUBE_CONFIG` within secret envrionment in Gitlab. (Settings -> CI/CD -> Variables)

## Setup Traefik

Below is probably not the correct way to setup traefik on kubernetes read this: `https://docs.traefik.io/v1.3/user-guide/kubernetes/`

```
helm install traefik --values traefik.yaml stable/traefik
kubectl describe svc traefik --namespace default | grep Ingress | awk '{print $3}'
```
Add in an A record into Route53 to the ELB created above and point it to `*`. Visit `https://traefik.cluster.playonmedia.com/`

### Install WordPress under Traefik

```
helm install wordpress stable/wordpress --set ingress.enabled=true --set ingress.hostname=wordpress.cluster.playonmedia.com
```

```
helm install wordpress --values wordpress.yaml stable/wordpress
```