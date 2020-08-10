
# Intro
That code is able to deploy Traefik as an Ingress controller on the Kubernetes cluster. In that example, I use K3S - a lightweight Kubernetes version. The cluster installation is pretty straightforward and has been described below. In the production, the environment considers automating that process and provide a highly available cluster with more than a master node. Please note that the K3S cluster had only one master node and should be used as production environment. 

## Creating K3S cluster
The cluster consisting of 3 nodes will be created manually. I assume that you have access to at least 3 virtual machines and can ssh to them. 

### node1

```ssh root@fist-node -p22101```

```  curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="server --no-deploy traefik --node-name node1" sh -  ```

Read node token that is going to be used to join another nodes to the cluster

``` cat /var/lib/rancher/k3s/server/node-token ```
### node2 

``` ssh root@second-node -p22101```

```curl -sfL https://get.k3s.io | K3S_URL=https://myserver:6443 K3S_TOKEN=mynodetoken INSTALL_K3S_EXEC="agent  --node-name node2" sh -```

Repeat that steps on a third node. 

### Add credentails to local kube.config

Copy the file `/etc/rancher/k3s/k3s.yaml` to your local desktop to `~/.kube/config`

```kubectl get nodes``` should return that 3 nodes cluster is up and running

```kubectl get node --selector='!node-role.kubernetes.io/master'``` - check what are worker nodes
# Add label to the node 
kubectl label nodes <node-name> <label-key>=<label-value>

### Traefik deployment

Once your local environemnt is connected to the cluster you can start with deploying Traefik and sample app. 

The example use CRD and [Kuberntes IngressRoute](https://docs.traefik.io/routing/providers/kubernetes-crd/)

#### Deploy CRD

```kubectl apply -f 00-prepare```

#### Deploy Traefik.

Before moving forward, please update the access URL for dashboard, update matching rule in that [file](https://github.com/jakubhajek/traefik-kubernetescrd/blob/ab8c214b2dee1346bc2c7c36db9ef341ad1ef7f9/01-traefik/03-traefik-dashboard.yml#L30)

You can also update [secrets](https://github.com/jakubhajek/traefik-kubernetescrd/blob/ab8c214b2dee1346bc2c7c36db9ef341ad1ef7f9/01-traefik/04-secret.yml#L8) and use your own username password. In this example login credentails are following: admin:admin1234

If you test your installation, consider to staging Let's Encrypt environemnt. You can replace [certifcate resolver](https://github.com/jakubhajek/traefik-kubernetescrd/blob/ab8c214b2dee1346bc2c7c36db9ef341ad1ef7f9/01-traefik/03-traefik-dashboard.yml#L39) to staging environment, see the config Traefik config file and [available resources.](https://github.com/jakubhajek/traefik-kubernetescrd/blob/ab8c214b2dee1346bc2c7c36db9ef341ad1ef7f9/01-traefik/02-traefik-config.yml#L43). Update also your email address. 

```kubectl apply -f 01-traefik```

#### Deploy test application

The test application containts frontend and backend written in NodeJS. This is just example to show how you can router incoming HTTP requests via Traefik to Nginx and to NodeJS backend. 


Update [the access URL](https://github.com/jakubhajek/traefik-kubernetescrd/blob/0f859392c5ecbe4e275bfc84d8b90461ede49d10/02-nginx-node-app/03-ingressroute.yml#L11) where you test application will be deployed. 

Please note that Nginx is just another layer to present multi-tier application. Normally, in that case you don't need to have another web server that acts as reverse proxy - just use Traefik for that purpose

```kubectl apply -f 02-nginx-node-app```

#### Canary deployment with Traefik and 

Canay deployment use another example that just presents HTTP headers. There two version of the app: V1 and V2 and differences between tham are only version header. 

Traefik use two weightedservices and route HTTP requestes between tham using [weights](https://github.com/jakubhajek/traefik-kubernetescrd/blob/fa194c0d71233741841901c52275baa80a823338/03-canary-deployment/02-canary-webapp-V2/ingress-canary.yml#L8) defined in that TraefikService. 

The first step is again related to [access URL](https://github.com/jakubhajek/traefik-kubernetescrd/blob/fa194c0d71233741841901c52275baa80a823338/03-canary-deployment/01-webapp-V1/02-ingressroute.yml#L12) to your application. 


Deploy the application V1 and check if it works correctly by accessing the URL.

```kubectl apply -f 03-canary-deployment/01-webapp-V1```

You should see the headers and nice title that request reached V1 app.

Now, you proceed with canary deployment using Traefik. More information about canary approach you can find in my recording: [Canary Deployment with Traefik and K3S](https://www.cometari.com/video/canary-deployment-with-traefik-and-k3s)

Deploy the second version of the app: V2

```kubectl apply -f 03-canary-deployment/01-webapp-V2```

Now, you have two services deployed: V1 and V2 and incoming HTTP traffic is balanced according to [weights](https://github.com/jakubhajek/traefik-kubernetescrd/blob/fa194c0d71233741841901c52275baa80a823338/03-canary-deployment/02-canary-webapp-V2/ingress-canary.yml#L8) defined in TraefikService. In the example we use 3 and 1, try to modify them and deploy the app once again. 
If you access the app you should be randomly redirected between V1 and V2.

#### How to reach webappp-V2 ?

The V2 Ingressroute adds another matching rule and specific headers defined as Middleware. If you add header to your request Traefik will use HTTP HOST and header in your request to route traffic directly to V2. This is defined in that [middleware](https://github.com/jakubhajek/traefik-kubernetescrd/blob/fa194c0d71233741841901c52275baa80a823338/03-canary-deployment/02-canary-webapp-V2/middleware.yml#L1)

Now, you can use following command and you will be always redirect to V2. You can use that approach to execute test and validate that your app (V2) works correctly. 

```curl --silent -iv -H "X-Canary-header: knock-knock" https://<your_application_url> | grep version -A2 -B2```

You should also notice a header in HTTP response, that means that your requests hits V2. 

If you have an questions concerning that example or you are interested in production implemenation of similar stack drop me a line. See you :)

