# Azure Lab Builders - Building an Azure Red Hat OpenShift Lab

## Various Links

- [Red Hat Cloud Experts](https://cloud.redhat.com/experts/)
- [Azure Front Door with ARO](https://cloud.redhat.com/experts/aro/frontdoor/)
- [Azure Arc with ARO](https://cloud.redhat.com/experts/aro/azure-arc-integration/)
- get your [pull secret](https://console.redhat.com/openshift/install/azure/aro-provisioned)

## AAD Integration with ARO

- described in MSFT [docs](https://learn.microsoft.com/en-us/azure/openshift/configure-azure-ad-ui)
- make your user a cluster-admin through a ClusterRoleBinding
- you've done it correctly if you can login and see the administrator view (like you would as kubeadmin)

![Cluster Role Binding](./images/aad-clusterrolebinding.png "Cluster Role Binding")

## Deploy Smoothies Rating App

### Deploy MongoDB

1. `oc new-project smoothies`
2. Deploy MongoDB

```
oc new-app bitnami/mongodb \
  -e MONGODB_USERNAME=ratingsuser \
  -e MONGODB_PASSWORD=ratingspassword \
  -e MONGODB_DATABASE=ratingsdb \
  -e MONGODB_ROOT_USER=root \
  -e MONGODB_ROOT_PASSWORD=ratingspassword
```

3. Our service will be at "mongodb.<project name>.svc.cluster.local"

### Deploy the Ratings API

1. Deploy the Ratings API

```
oc new-app https://github.com/mrhoads/azure-lab-builders-openshift --strategy=source --name=rating-api
```

Note that if you go to the console after running this, you'll see the container being built

2. Set the environment variable for the MongoDB URI

![MongoDB Env](./images/mongo-environment.png "MongoDB Environment")

It can also be added with `oc set env deploy/rating-api MONGODB_URI=mongodb://ratingsuser:ratingspassword@mongodb.smoothies.svc.cluster.local:27017/ratingsdb`

Verify that you see a log for the rating-api pod like, "'CONNECTED TO mongodb://ratingsuser:ratingspassword@mongodb.prep-smoothies.svc.cluster.local:27017/ratingsdb'"

3. Change the service port to 3000

If you were to try `oc port-forward svc/rating-api 8080:8080` and then tried to curl localhost:8080, it'd fail.  The API isn't there.  Change it as in the image below

![API Service Port](./images/api-service-port.png "API Service Port")

### Deploy the Ratings Frontend

1. See the different Dockerfile and Footer.vue from the [original workshop page](https://microsoft.github.io/aroworkshop/)

2. Deploy rating-web with `oc new-app https://github.com/mrhoads/azure-lab-builders-openshift-ratings-web --strategy=docker --name=rating-web`

Note the different strategy here.  If you look in the repo, it contains a Dockerfile

Consider looking at setting up a webhook to trigger builds when changes are made to GitHub.  Run:
`oc get bc/rating-web -o=jsonpath='{.spec.triggers..github.secret}'` to get the secret value and...
`oc describe bc/rating-web` to get the webhook URL

Then configure that webhook in your GitHub repo, being sure to set the content type to *application/json*

Make a commit and look for a new build.

3. Set the environment variable so rating-web knows where the API is

`oc set env deploy rating-web API=http://rating-api:3000`

4. Expose the rating-web service

`oc expose svc/rating-web`

5. Verify the ratings app is accessible

Note that in the topology view, it's now possible to go to the app via the exposed route

![Expose Web Route](./images/rating-web-open-url.png "Expose Web Route")

### Deploy the Ratings App via an Ingress and Lets Encrypt TLS Certificate

Here I'm basically following what the team of ARO GBBs documented [here](https://cloud.redhat.com/experts/aro/additional-ingress-controller/)

1. Create variables for Lets Encrypt

```
DOMAIN=alb.k8s.5amworkshop.com
EMAIL=mysupersecretemail@something.com
SCRATCH_DIR=/tmp/aro
```

2. Use certbot to get cert and use DNS as the challenge

```
certbot certonly --manual \
  --preferred-challenges=dns \
  --email $EMAIL \
  --server https://acme-v02.api.letsencrypt.org/directory \
  --agree-tos \
  --manual-public-ip-logging-ok \
  -d "*.$DOMAIN" \
  --config-dir "$SCRATCH_DIR/config" \
  --work-dir "$SCRATCH_DIR/work" \
  --logs-dir "$SCRATCH_DIR/logs"
```

3. Create a secret in the openshift-ingress NS

```
oc create secret tls alb-k8s-5amworkshop-tls \
  -n openshift-ingress \
  --cert=$SCRATCH_DIR/config/live/$DOMAIN/fullchain.pem \
  --key=$SCRATCH_DIR/config/live/$DOMAIN/privkey.pem
```

4. Create an ingress controller

```
cat <<EOF | oc apply -f -
apiVersion: operator.openshift.io/v1
kind: IngressController
metadata:
  name: alb-prep-ratings
  namespace: openshift-ingress-operator
spec:
  domain: smoothies.alb.k8s.5amworkshop.com
  nodePlacement:
    nodeSelector:
      matchLabels:
        node-role.kubernetes.io/worker: ""
  routeSelector:
    matchLabels:
      type: custom
  defaultCertificate:
    name: k8s-5amworkshop-wildcard-custom-tls
  httpEmptyRequestsPolicy: Respond
  httpErrorCodePages:
    name: ""
  replicas: 3
EOF
```

5. Create TLS Route

```
cat << EOF | oc apply -f -
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  labels:
    app: hello-openshift
    app.kubernetes.io/component: hello-openshift
    app.kubernetes.io/instance: hello-openshift
    type: custom
  name: smoothies-openshift-tls
spec:
  host: smoothies.alb.k8s.5amworkshop.com
  port:
    targetPort: 8080-tcp
  tls:
    termination: edge
    insecureEdgeTerminationPolicy: Redirect
  to:
    kind: Service
    name: rating-web
EOF
```

6. Get the External IP of the Load Balancer used for the ingress

`oc get -n openshift-ingress svc router-alb-prep-ratings`

!["External IP of Rating Web"](./images/svc-external-ip.png "External IP of Rating Web")

7. Create an A Record

In Azure DNS or other DNS provider, create an A record with that IP address

!["Creating A Record](./images/a-record.png "A Record")

8. Visit your app

It should now have a valid TLS certificate


## Operators

Expore the various Operators available.  I've deployed the AWX operator

!["AWX Operator"](./images/awx-operator.png)

Through that, I can use the various APIs that operator provides to deploy something.  In my case, I've already created an "AWX" instance.

I've exposed this via an ingress controller and have a Lets Encrypt certificate on it.  Let's take a look at some of what this particular operator exposes through the console like:
- setting passwords and other properties of the instance
- setting the hostname

## Using Log Analytics with ARO

The following section is based on what Red Hat GBBs [describe here](https://cloud.redhat.com/experts/aro/clf-to-azure/)

1. I've already created a Log Analytics Workspace and need the key and workspace ID from it:

```
WORKSPACE_ID=$(az monitor log-analytics workspace show \
 -g <RG> -n <Workspace Name> \
 --query customerId -o tsv)
 ```

 ```
 SHARED_KEY=$(az monitor log-analytics workspace get-shared-keys \
 -g <RG> -n <Workspace Name> \
 --query primarySharedKey -o tsv)
 ```

2. Create a new project

`oc project aro-clf-am`

3. Create new namespaces

```
kubectl create ns openshift-logging
kubectl create ns openshift-operators-redhat
```

4. Add the appropriate Helm repo

`helm repo add mobb https://rh-mobb.github.io/helm-charts/`
 
 5. Update Helm repos

 `helm repo update`

 6. Deploy Elasticsearch Operators

 ```
 helm upgrade -n $NAMESPACE clf-operators \
 mobb/operatorhub --version 0.1.1 --install \
 --values https://raw.githubusercontent.com/rh-mobb/helm-charts/main/charts/aro-clf-am/files/operators.yaml
```

7. Deploy the Custom Log Forwarder

```
helm upgrade -n $NAMESPACE clf \
 mobb/aro-clf-am --install \
 --set "azure.workspaceId=$WORKSPACE_ID" --set "azure.sharedKey=$SHARED_KEY"
```

8. Wait!

In ~10 minutes you should see a new table, *openshift_CL*, in your workspace.