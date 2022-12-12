# Timescale Cloud Public API - Alpha Version

This is the documentation for the alpha version of [Timescale Cloud]'s public
API.

**It is located at https://console.cloud.timescale.com/api/query**

The API is a graphQL API, and requires authentication in the form of a token.

- [Timescale Cloud Public API - Alpha Version](#timescale-cloud-public-api---alpha-version)
  - [Authentication](#authentication)
  - [Available Endpoints](#available-endpoints)
    - [Queries](#queries)
      - [Get Plans and Products](#get-plans-and-products)
      - [Get Service(s)](#get-services)
      - [Get Service Deployment Status](#get-service-deployment-status)
      - [Get Service Logs](#get-service-logs)
      - [Get Resource Metrics](#get-resource-metrics)
    - [Mutation](#mutation)
      - [Create Service, including forks](#create-service-including-forks)
      - [Resume / Pause service](#resume--pause-service)
      - [Resize instance](#resize-instance)
      - [Delete service](#delete-service)

## Authentication

If you are a part of this alpha, a specific token was issued for your project and communicated to you.
This token has a fixed duration for now and will be valid for the entire duration of the alpha.

It is to be used as a Bearer token in the Authentication HTTP header as follows :

`{"Authentication":"Bearer $YOUR_TOKEN"}`

Reach out to customer support for assistance getting your token, or if you can't authenticate to our API.

## Available Endpoints

We hereby give examples of the most exhaustive payload possible on all endpoints available on this API.
GraphQL APIs returns values that are queried for only, allowing you to customize the response and hide unwanted fields.
For more information, read the [GraphQL Documentation].

*Note: The alpha API doesn't support multinodes.*

### Queries

In the graphQL framework, queries only allow to read values.

#### Get Plans and Products

`products` gets all the different plans we offer, grouped by products : TimescaleDB compute, TimescaleDB storage, Instance VPC attachement, vanilla PostgreSQL compute, vanilla PostgreSQL storage

``` graphql
query {
    products {
        id
        name
        description
        plans {
            id
            productId
            price
            milliCPU
            ramGB
            storageGB
            regionCode
        }
    }
}
```

#### Get Service(s)

`getService` will return service details for given project_id/service_id.

For the return value, see all available service fields [here](#service-object-fields).

``` graphql
query {
    getService (data:{
        serviceId:"$YOUR_SERVICE_ID",
        projectId:"$YOUR_PROJECT_ID"
    }) {
        name
        id
        ...
    }
}
```

`getAllServices` will return all the services for given project_id, with the
same available response.

For the return value, see all available service fields [here](#service-object-fields).


``` graphql
query {
    getAllServices (projectId:"$YOUR_PROJECT_ID") {
        name
        id
        ...
    }
}
```

####  Get Service Deployment Status

More lightweight, `getServiceDeploymentStatus` allows to query for the services' current deployment status only, returning the appropriate string.

``` graphql
query {
    getServiceDeploymentStatus (data:{
        serviceId:"$YOUR_SERVICE_ID",
        projectId:"$YOUR_PROJECT_ID"
    }) {}
}
```

Possible values are following

```
QUEUED
DELETING
CONFIGURING
READY
DELETED
UNSTABLE
PAUSING
PAUSED
RESUMING
UPGRADING
OPTIMIZING
```

#### Get Service Logs

`getServiceLogs` returns an array of strings, each one representing one log line for queried instance. This call will pull the most recent payload of logs *from the current 24hr window*. It returns a maximum of 500 log entries.

`serviceOrdinal` is an optional parameter, necessary in case of high availability (i.e. replicas), to query for one specific replica.

``` graphql
query {
    getServiceLogs (data:{
        serviceId:"$YOUR_SERVICE_ID",
        projectId:"$YOUR_PROJECT_ID",
        serviceOrdinal:0
    }) {}
}
```

#### Get Resource Metrics

`getLastServiceResourceMetrics` will get the last resource usage of a service.

`getServiceResourceMetrics` will get the resource usage of a service on a defined period.

`serviceOrdinal` is an optional parameter, necessary in case of high availability (i.e. replicas), to query for one specific replica.

The response will be an array of measurements, representing CPU, RAM & storage usage at specific points in time.

``` graphql
query {
    getLastServiceResourceMetrics (data:{
        serviceId:"$YOUR_SERVICE_ID",
        projectId:"$YOUR_PROJECT_ID",
        serviceOrdinal:0
    }) {
        memoryMB
        storageMB
        milliCPU
        Time
    }
}
```

``` graphql
query {
    getServiceResourceMetrics (data:{
        serviceId:"$YOUR_SERVICE_ID",
        projectId:"$YOUR_PROJECT_ID",
        start: "2022-01-01T00:00:00.000000Z",
        end: "2022-01-31T00:00:00.000000Z",
        bucketSeconds: 30,
        serviceOrdinal:0
    }) {
        memoryMB
        storageMB
        milliCPU
        Time
    }
}
```


### Mutation

In the graphQL framework, mutations allow modifying values.

#### Create Service, including forks

`createService` allows creating all types of services.

Available `regionCode`s are
```
us-east-1
us-west-2
eu-west-1
eu-central-1
ap-southeast-2
```

`vpcId` can hold the ID of a VPC created beforehand, to attach it to the service
at start up.

`enableStorageAutoscaling` will allow your instance to scale automatically to
prevent full disk errors.

`forkConfig` might hold another services' ID, to copy that service's data to the
instance to be created. It must belong to the same project_id for now.

For a single node service, `type` must be `TIMESCALEDB`, and `resourceConfig` will hold the instance specifications.

The response will hold the instances initial password, and the eventual service.
For the service return value, see all available service fields [here](#service-object-fields).

Single node example, with fork:

``` graphql
mutation {
    createService(data:{
        projectId:"$YOUR_PROJECT_ID",
        name:"$SERVICE_NAME",
        type:TIMESCALEDB,
        vpcId:1,
        regionCode: "us-east-1",
        enableStorageAutoscaling: false,
        resourceConfig:{
            milliCPU: 250,
            storageGB: 10,
            memoryGB: 1,
            replicaCount: 0
        },
        forkConfig:{
            projectID:"$YOUR_PROJECT_ID",
            serviceID:"$SOURCE_SERVICE_ID"
        }
    }){
        initialPassword
        service {
            id
            projectId
            ...
        }
    }
}
```

#### Resume / Pause service

`toggleService` allows resuming or pausing your service.

`status` is the goal status and can take two values: `ACTIVE` and `INACTIVE`. If
the instance is already in desired status, an error will be returned.

For the service return value, see all available service fields
[here](#service-object-fields).

``` graphql
mutation {
    toggleService(data:{
        serviceId:"$YOUR_SERVICE_ID",
        projectId:"$YOUR_PROJECT_ID",
        status:"ACTIVE"
    }){
        id
        projectId
        ...
    }
}
```


#### Resize instance

`resizeInstance` allows resizing CPU, RAM, and Storage for your service.

*Note: Storage cannot be downsized.*

``` graphql
mutation {
    resizeInstance(data:{
        serviceId:"$YOUR_SERVICE_ID",
        projectId:"$YOUR_PROJECT_ID",
        config: {
            milliCPU: 250,
            storageGB: 10,
            memoryGB: 1
        }
    }){}
}
```

#### Delete service

`deleteService` allows deleting your service.

For the service return value, see all available service fields
[here](#service-object-fields).

``` graphql
mutation {
    deleteService(data:{
        serviceId:"$YOUR_SERVICE_ID",
        projectId:"$YOUR_PROJECT_ID"
    }){
        id
        projectId
        ...
    }
}
```

## Service object fields

Service is a complex object. All the following fields are available:

``` graphql
{
    name
    id
    projectId
    type
    isReadOnly
    created
    regionCode
    spec {
        ... on TimescaleDBServiceSpec {
        hostname
        username
        port
        defaultDBName
        }
    }
    resources {
        id
        spec {
            ... on ResourceNode {
                milliCPU
                memoryGB
                storageGB
            }
        }
    }
    status
    vpcId
    vpcEndpoint {
        ... on Endpoint {
        projectId
        serviceId
        vpcId
        host
        port
        }
    }
    autoscaleSettings {
        enabled
        upper_limit_gb
    }
    metrics {
        ... on ResourceMetrics {
        memoryMB
        storageMB
        milliCPU
        }
    }
    primaryOrdinal
    replicaOrdinals
    replicaStatus
}
```

[Timescale Cloud]: https://console.cloud.timescale.com/
[GraphQL Documentation]: https://graphql.org/learn/
