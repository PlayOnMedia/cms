# AWS / KOPS / HELM

## Requirements

* **Kubectl:** https://kubernetes.io/docs/tasks/tools/install-kubectl/
* **KOPS:** https://github.com/kubernetes/kops
* **Helm:** https://helm.sh/docs/using_helm/
* **AWS CLI:** https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-install.html
* **AWS Access Creds:** https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-configure.html
* **AWS S3 Bucket:** https://docs.aws.amazon.com/cli/latest/reference/s3api/create-bucket.html

Setup AWS credentials locally (~/.aws/credentials) to create the s3 bucket/run KOPS commands
```
[default]
aws_access_key_id = XXXXXXXXXXXXXXXX
aws_secret_access_key = XXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
```

Create a bucket to store the KOPS state.
```
aws s3api create-bucket --bucket kops-state-playonmedia --region us-east-1
aws s3api put-bucket-versioning --bucket kops-state-playonmedia --versioning-configuration Status=Enabled
export KOPS_STATE_STORE=s3://kops-state-playonmedia
```

## Create Cluster

### With Subdomain
https://github.com/kubernetes/kops/blob/master/docs/getting_started/aws.md

```
ID=$(uuidgen) && aws route53 create-hosted-zone --name cluster.playonmedia.com --caller-reference $ID | jq .DelegationSet.NameServers
aws route53 list-hosted-zones | jq '.HostedZones[] | select(.Name=="playonmedia.com.") | .Id'
touch subdomain.json
aws route53 change-resource-record-sets --hosted-zone-id Z100Q9OTKLC6GD --change-batch file://subdomain.json
dig ns cluster.playonmedia.com
```


Run locally or add add to `vi ~/.bash_profile` or equiv

This allows us to avoid using `--state=s3://kops-state-playonmedia` with `kops` `create`, `update`, `validate`, and `delete` commands.

```
export KOPS_STATE_STORE=s3://kops-state-playonmedia
```

```
kops create cluster --name=cluster.playonmedia.com --zones=us-east-1a --node-count=2 --node-size=t2.micro --master-size=t2.micro --yes
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
kubectl create serviceaccount --namespace kube-system tiller
kubectl create clusterrolebinding tiller-cluster-rule --clusterrole=cluster-admin --serviceaccount=kube-system:tiller
helm init --service-account tiller
```

### Installing a WordPress example chart
```
helm install --name wordpress stable/wordpress
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
helm install --name traefik --values traefik.yaml stable/traefik
kubectl describe svc traefik --namespace default | grep Ingress | awk '{print $3}'
```
Add in an A record into Route53 to the ELB created above and point it to `*`. Visit `https://traefik.cluster.playonmedia.com/`

### Install WordPress under Traefik

```
helm install --name wordpress stable/wordpress --set ingress.enabled=true --set ingress.hostname=wordpress.cluster.playonmedia.com
```

```
helm install --name wordpress --values wordpress.yaml stable/wordpress
```