# How to install TeslaMate in an Azure cluster

This is a guide to help you install [TeslaMate](https://github.com/adriankumpf/teslamate) on an Azure Kubernetes cluster.

----
## Disclaimer!

__You should be very careful with blindly following any guide on the internet, including this one!__ Try and understand what the code does before you execute it. I've used this approach myself to create a cluster with TeslaMate and Grafana installed and naturally tried to secure it as best as possible. However, I can't guarantee it will stand the test of time..

__I assume you have some knowledge of the Azure portal__, because describing that interface is just to painful. I'll try and be as accommodating as I can, but there are limits :)

__All script provided are powershell scripts__, but you should be able to translate them to any shell and azure cli tool you want.

__There will be costs involved if you follow this guide!__

__There will be costs involved if you follow this guide!__

__There will be costs involved if you follow this guide!__

---

Any remarks, suggestions, niceties or much-needed changes? Make an issue/pull request!

---

## Prerequisites
__You should have a kubernetes cluster set up.__ In Azure a single cheap node should suffice. I use a single node pool with kubernetes version 1.17.0 containing a single standard_b2s node. I'm fairly certain a standard_B1ms will work as well, though I'd recommend not to get any lower. Also, some persistence storage is nice.

I've opted to use RBAC in the cluster, which complicates some stuff. This guide will assume you've done this as well and you want a working kubernetes dashboard. So it will be laced with some stuff to make everything work with RBAC.

__You also should have the following tools installed__. (The following links go to the official websites)
- [Azure CLI](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest) to manage Azure resources programmatically.
- [Helm v3](https://helm.sh/docs/intro/install/) a deployment manager we will use to intall TeslaMate and Grafana
- [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/) the manage the kubernetes cluster.

## Configuring the tooling and the cluster
The first thing we will do is assign a public IP address to your cluster and bind it to a domain name. This will ensure we get a reliable way of contacting any service we want inside the cluster (like TeslaMate). 

We need to log in first
```powershell
# Ensure the Azure cli will use your account to manage resources
az login 
# Will set the kubeconfig of kubectl, so we can interact with the cluster
$clusterName = "default-cluster" #replace this with your cluster name in Azure
$resourceGroup = "default" #replace this with the resource group where your cluster resides in Azure
$clusterResourceGroup = "MC_default_default-cluster_westeurope" #replace this with the resource group that was created for your cluster
az aks get-credentials -n $clusterName -g default $resourceGroup
```

Next we will ensure the kubernetes dashboard will work nicely with RBAC by giving the clusterUser user (which azure uses) some more rights.
```powershell
kubectl create clusterrolebinding kubernetes-dashboard -n kube-system --clusterrole=cluster-admin --serviceaccount=kube-system:kubernetes-dashboard
kubectl create clusterrolebinding clusterUser -n kube-system --clusterrole=cluster-admin --user=clusterUser
```
After this you should be able to access the kubernetes dashboard using the following command.:
```powershell
az aks browse --resource-group $resourceGroup --name $clusterName
```
This way you can check at any step the status of your cluster and also any errors in deployments.


### Assigning a domain to your cluster

Now we will crate a public static IP adress which we can later assign to a DNS record so when you go to your domain you get routed to your cluster.
```powershell
az network public-ip create --resource-group $clusterResourceGroup  --name PublicIP --sku Basic --allocation-method static #create the ip adress
#wait a bit for azure to provision the ip adress
 az network public-ip list --query "[?ipAddress!=null]|[?contains(name, 'PublicIP')].[ipAddress]" --output tsv #returns the ip adress
 $ipAdress = '10.0.0.1' #replace this with your created ip adress
```
Inside the Azure Portal create a DNS name inside the resource group of your cluster ($clusterResourceGroup). Add a new A record * with the ip adress as your value. You can do this manually or with the command below. Note: if you don't already have a dns zone, create one using this guide: https://docs.microsoft.com/azure/dns/dns-delegate-domain-azure-dns.

```powershell
$dnsZoneName = "your-zone-name.contoso.com"

az network dns record-set a add-record `
    --resource-group $clusterResourceGroup `
    --zone-name  $dnsZoneName `
    --record-set-name * `
    --ipv4-address $ipAdress
```

We're now ready to add nginx-ingress to your cluster. It will provide a way to manage external access to your services as well as a way for us to secure it later on.
We also want to put it behind https so we need a valid certificate, which we will provision using letsencrypt. Furthermore, we want to secure any future service, so we're going to be using a oauth2 proxy which will use github.com to let people sign in (you can use other providers as well, such as google).

```powershell
kubectl create namespace ingress-basic # we create a namespace to put the controller in

# we install nginx into the cluster
helm upgrade nginx stable/nginx-ingress `
    --install `
    --namespace ingress-basic `
    --set controller.replicaCount=1 `
    --set controller.nodeSelector."beta\.kubernetes\.io/os"=linux `
    --set defaultBackend.nodeSelector."beta\.kubernetes\.io/os"=linux `
    --set controller.service.loadBalancerIP="$ipAdress"

# we're going to install the certification managager to get our certificates
kubectl apply --validate=false -f https://raw.githubusercontent.com/jetstack/cert-manager/release-0.12/deploy/manifests/00-crds.yaml --namespace ingress-basic

# Label the ingress-basic namespace to disable resource validation
kubectl label namespace ingress-basic certmanager.k8s.io/disable-validation=true

# Add the Jetstack Helm repository
helm repo add jetstack https://charts.jetstack.io

# Update your local Helm chart repository cache
helm repo update

# Install the cert-manager Helm chart
helm install cert-manager --namespace ingress-basic --version v0.12.0 jetstack/cert-manager --set ingressShim.defaultIssuerName=letsencrypt --set ingressShim.defaultIssuerKind=ClusterIssuer

# open up cluster-issuer,yaml and replace <<YOUR_EMAILADRESS>> with your email. Note this will be public, so maybe a spam mail?
#next, we're going to create the cluster certificate issuer, which will enable us to request certificates from letsencrypt
kubectl apply -f cluster-issuer.yaml --namespace ingress-basic
```

### Securing services exposed using oauth2
 We're going to use oauth2-proxy to secure any services we expose. You can find more documentation over at https://pusher.github.io/oauth2_proxy/. I'm going to use Github.com as a provider, but you can choose from a range of others if you want: https://pusher.github.io/oauth2_proxy/auth-configuration

 First we need an application registered with Github. See https://developer.github.com/apps/building-oauth-apps/creating-an-oauth-app/ on how to do this. 

 Next we're going to install the proxy. For this we're going to need a file with the configuration oauth2-proxy-config.yaml. Open up the file and replace the text ```<<LIKE_THIS>>``` with the proper values.

 You also need to make a choice at this point. If you want to use [TeslaMate wake up hints](https://teslamate.readthedocs.io/en/latest/configuration/sleep.html#providing-wake-up-hints-to-teslamate) you'll need a way to circumvent the github login screen (for instance for tasker). We can do this by commenting out the code inside the file and supply a custum username and password.
 
 __Note this will make the install slightly less secure and you should really use a sufficiently long password__ (there is no bruteforce protection! so probably upwards of 56 chars).
  
The username and password entry can be created on linux using the following command ```htpasswd -sbc passwords <username> <password>```.This will create a file named passwords which will contain an entry like: ```username:{SHA}W6ph5Mm5Pz8GgiULbPgzG37mj9g=```

On windows, it can be created using the following script
```powershell
$password = "some long password"
$username = "some username"


$hash = [System.Security.Cryptography.SHA1]::Create()
$bytes = [System.Text.Encoding]::UTF8.GetBytes($password)
$byteString = [System.Convert]::ToBase64String($hash.ComputeHash($bytes))

"$username"+':{SHA}'+"$byteString"
```


Now we're ready to install the proxy:
```powershell
$cookieSecret = "someGeneratedString" #generate a random string to use as a secret, you can use the next commented out line to generate a secret if you want:
# -join ((48..57) + (65..90) + (97..122) | Get-Random -Count 56 | % {[char]$_})
$githubClientId = "some_id_provided_by_github" # see https://developer.github.com/apps/building-oauth-apps/creating-an-oauth-app/ for registering an application with github
$githubSecret = "some_secret_provided_by_github"
kubectl -n default create secret generic oauth2-proxy-creds `
    --from-literal=cookie-secret=$cookieSecret `
    --from-literal=client-id=$githubClientId `
    --from-literal=client-secret=$githubSecret

#install oauth2-proxy
helm upgrade oauth2-proxy --install stable/oauth2-proxy `
    --reuse-values `
    --values oauth2-proxy-config.yaml
```


### Install Teslamate and Grafana

```powershell
#adds the helm repository containing the helm chart for teslamate
helm repo add billimek https://billimek.com/billimek-charts/ 

#install teslamate
helm upgrade teslamate billimek/teslamate `
    --install `
    --set virtualHost="https://teslamate.$dnsZoneName"


#install grafana
#note: I've handcrafted this values.yaml file. It contains some default credentials you might want to change...
helm install -f .\values.yaml grafana stable/grafana 

#gets the created password for the admin account of grafana which you'll need later on to log in (first time, after which you should create a new user!)
[System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String($(kubectl get secret --namespace default grafana -o jsonpath="{.data.admin-password}")))
```

### Expose Teslamate and Grafana so you can access it from anywhere
The ingress.yml file contains the routes to set up. Open this file and replace ``<<YOUR_DOMAIN>>`` with your domain. After run the following commands

```powershell
kubectl apply -f ingress.yaml
```

### Some additionals
You might want to log in to ```teslamate.<<YOUR_DOMAIN>>``` now using your Tesla account.
Also, this is the time to log in to ```grafana.<<YOUR_DOMAIN>>``` to change create that other user + password and disable the default admin account... 
