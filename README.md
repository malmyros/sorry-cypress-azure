# Sorry Cypress

Sorry Cypress is an open source solution for Cypress that provides
a free dashboard, and a load balancer in order to distribute tests
across different machines.

To get sorry-cypress running we need the following services:

- API
- Dashboard
- Director
- MongoDB
- Ingress Route
- Certificates

### Documentation

Documentation for sorry-cypress can be found here:

- [Documentation](https://docs.sorry-cypress.dev/)
- [GitHub](https://github.com/sorry-cypress/sorry-cypress)

### Installation

To install sorry-cypress in kubernetes on azure we need to log in the portal
and then click on the terminal icon on the top.

When the terminal has loaded we need to get the credentials before we can run the
required commands.

```
az aks get-credentials --resource-group clarity-go-resource-group --name <AKS_CLUSTER_NAME>
```

### API

API Service is a simple GraphQL wrapper that exposes a convenient way to query the data stored by Director.
The service is only required as an interface for the Web Dashboard, but can be used to query the database 
and describe the internal data models.

```
kubectl apply -f cypress-api.yml
```

To delete a run visit the API and set the following:

- Query Panel
```
mutation deleteRun($runId: ID!) {
  deleteRun(runId: $runId) {
    success
    message
    runIds
    __typename
  }
}
```

- Variables Panel

Can be found from the url in the cypress dashboard
```
{
  "runId": "<RUN_ID>"
}
```

### Dashboard

The dashboard allows end-users to interact with sorry-cypress using a browser and to:
track test runs progress

- browser test results, videos, and failures screenshots
- set projects configuration like WebHooks, Slack, MS Teams and GitHub integration
- create and delete entries (projects, runs)

```
kubectl apply -f cypress-dashboard.yml
```

### Director

The director service is responsible parallelization and coordination of test runs.

```
kubectl apply -f cypress-director.yml
```

### MongoDB

```
kubectl apply -f mongo-db.yml
```

Once MongoDB is installed we need to log into our pod and create a database.
This is where our runs are going to be saved.

First log in the pod:

```
kubectl get pods
kubectl exec -it <POD-NAME> -- /bin/bash 
```

Now run the following to create a default database

```
mongo
use sorry-cypress
```

### Ingress Route

To install it we need to do the following:

### Add the ingress-nginx repository with helm

```
helm repo add ingress https://kubernetes.github.io/ingress-nginx
```

Now we can create our nginx ingress controller for our namespace and apply
the public static IP address we created earlier. The ingress controller will be
configured using an ingress route to direct traffic to each pod in the cluster.

```
helm install nginx-ingress ingress-nginx/ingress-nginx \
    --namespace default \
    --set controller.replicaCount=2 \
    --set controller.nodeSelector."beta\.kubernetes\.io/os"=linux \
    --set defaultBackend.nodeSelector."beta\.kubernetes\.io/os"=linux \
    --set controller.admissionWebhooks.patch.nodeSelector."beta\.kubernetes\.io/os"=linux
```

### View the status of the controller running

```
kubectl get services -o wide -w nginx-ingress-ingress-nginx-controller
```

### Create the Ingress Controller

```
kubectl apply -f <ENVIRONMENT>-ingress-route.yml
```

### Certificates

Before we start applying manifest files for creating our resources 
in our cluster we need to install the Cert Manager.

```
# Label the cert-manager namespace to disable resource validation
kubectl label namespace default cert-manager.io/disable-validation=true

# Add the Jetstack Helm repository
helm repo add jetstack https://charts.jetstack.io

# Update your local Helm chart repository cache
helm repo update

# Install the cert-manager Helm chart
helm install cert-manager jetstack/cert-manager \
  --namespace default \
  --set installCRDs=true \
  --set nodeSelector."kubernetes\.io/os"=linux \
  --set webhook.nodeSelector."kubernetes\.io/os"=linux \
  --set cainjector.nodeSelector."kubernetes\.io/os"=linux
```

Next we need to install our cluster issuer which is going to be
responsible for generating the certificates for our domain and
sub-domains.

```
# Create a CA cluster issuer
kubectl apply -f cluster-issuer.yml
```

To generate a new certificate we can apply the following
configuration on our cluster, and the Cert Manager will 
handle everything else for us.

```
kubectl apply -f certificates.yml --validate=false
```

When we want to evaluate whether our certificate for SSL 
is configured correctly.

To do this we need to run:

```
kubectl get certificate -n default
```

### Azure pipeline

Finally here is an example of a job that will run sorry-cypress against 3 machines.

```
- job: RunCypress
        displayName: 'Run Cypress'
        strategy:
          parallel: 3
        pool:
          vmImage: $(vmImageName)
        steps:
          - task: NodeTool@0
            inputs:
              versionSpec: '14.17.3'
            displayName: 'Install Node.js'

          - script: |
              echo -e 'pcm.!default {\n type hw\n card 0\n}\n\nctl.!default {\n type hw\n card 0\n}' > ~/.asoundrc
            displayName: 'Set the sound card to null to prevent Cypress from timing out'

          - script: |
              npm i -D
            displayName: 'Install Dependencies'

          - script: |
              CYPRESS_API_URL="https://cypress-director.<DOMAIN>.com/" npx cy2-azure run --config-file cypress.dev.json --parallel --record --key <KEY> --ci-build-id $(tag) --group "Azure CI"
            displayName: 'Run Integration Tests'
            env:
              TERM: xterm
```
