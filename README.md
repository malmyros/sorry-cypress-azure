# Sorry Cypress

Sorry Cypress is an open source solution for Cypress that provides
a free dashboard, and a load balancer in order to distribute tests
across different machines.

To get sorry-cypress running we need the following services:

- API
- Dashboard
- Director
- MongoDB

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
