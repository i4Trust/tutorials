# Data Service Provider

This describes how to setup an NGSI-LD based data service provider within the i4Trust trusted data space with the 
example of a fictitious Packet Delivery Company.

The actual NGSI-LD data service is provided by an instance of the Orion Context Broker. The following instructions 
describe how to deploy all required components on Kubernetes using Helm charts.

The setup within an i4Trust data space is based on the environment of an NGSI-LD data service provider, as described 
in the [production-on-k8s](https://github.com/FIWARE/production-on-k8s/tree/main/NGSI-LD_Data-Provider) repository 
of the [FIWARE Foundation](https://www.fiware.org). In the following only the differences to the linked setup will 
be presented.

The whole environment will consist of the following components:
* MongoDB
* Orion Context Broker
* Elasticsearch
* API-Umbrella (PEP Proxy/PDP)
* Keyrock (Identity Provider)
* MySQL
* Activation Service
* Portal demo application

as depicted in the following diagram:

![Components](./img/components.png "Components")

Furthermore it is required that there is access to an iSHARE Satellite instance and an iSHARE-compliant Authorisation 
Registry. It is assumed that the data service provider is already registered at the iSHARE Satellite and that 
certificates, a private key and the EORI have been issued. 

In the following it is assumed that the components will be externally available via the domain `domain.org` and that the 
issued EORI is `EU.EORI.NLPACKETDEL`. 

This description provides examples of the [Helm values files](./values) which show the minimum configuration 
parameters to be set. Adapt these for your setup before proceeding with the instructions.

The helm charts of the FIWARE Generic enablers with all possible configuration values can be found here:
* [orion](https://github.com/FIWARE/helm-charts/tree/main/charts/orion)
* [API Umbrella](https://github.com/FIWARE/helm-charts/tree/main/charts/api-umbrella)
* [Keyrock](https://github.com/FIWARE/helm-charts/tree/main/charts/keyrock)

Helm charts of the i4Trust related components with all possible configuration values can be found here:
* [Activation Service](https://github.com/i4Trust/helm-charts/tree/main/charts/activation-service)
* [Packet Delivery Portal - Demo Application](https://github.com/i4Trust/helm-charts/tree/main/charts/pdc-portal)

Is is assumed that all components will be deployed within the namespace `provider`.
```shell
kubectl create ns provider
```


## Databases and Orion Context Broker

For the components of
* Orion Context Broker
* MongoDB
* MySQL
* Elasticsearch

there is no changed configuration compared to the instructions of the repository 
[production-on-k8s](https://github.com/FIWARE/production-on-k8s/tree/main/NGSI-LD_Data-Provider). Just follow the steps 
described in the linked repository.



## Keyrock

The Keyrock Identity Provider is required for storing the user accounts of the employees of the data service provider's 
company. In the experimentation framework example, it is needed in order that employees can login at the i4Trust 
Marketplace and create offerings on behalf of their company.

Modify the Keyrock [values file](./values/values-keyrock.yml) according to your needs and deploy the Keyrock Identity Provider. 
Make sure to setup an Ingress or OpenShift route in the values file for external 
access of the UI (e.g. https://keyrock.domain.org). Also note that for the moment a dedicated Keyrock build needs to be used until 
the i4Trust related changes have been officially released: `fiware/idm:i4trust-rc3`. The issued private key and certificate 
chain must be added in PEM format. 
Make sure to use the chart from this [branch](https://github.com/FIWARE/helm-charts/tree/i4trust/charts/keyrock) until 
the chart has been officially released.
```shell
# Chart not officially released yet
#helm repo add fiware https://fiware.github.io/helm-charts/
#helm repo update
# Use https://github.com/FIWARE/helm-charts/tree/i4trust/charts/keyrock instead
helm install -f ./values/values-keyrock.yml --namespace provider keyrock fiware/keyrock --version 0.0.3
```

In a browser open the Keyrock UI (e.g. https://keyrock.domain.org) and login with the admin credentials provided in 
the values file. Then users can be created by the Admin user, or users sign up on their own.



## API-Umbrella

API-Umbrella is used as Policy Enforcement Point (PEP) and Policy Decision Point (PDP).

First create the necessary database and user within the MongoDB. This requires to enter a shell within the MongoDB pod, 
starting the MongoDB client and perform the commands below for creating the user.
```shell
# Enter pod
kubectl exec -n provider --stdin --tty mongodb-xyz -- /bin/bash

# Start mongo client
mongo -u root     # (provide MongoDB root PW)

# Within mongo client:
> use api_umbrella
> db.createUser({user: "api_umbrella", pwd: "api-umbrella-password", roles: [{role: "readWrite", db: "api_umbrella"}]})
```

Now modify the [API Umbrella values file](./values/values-umbrella.yml) according to your setup and perform 
the deployment using Helm. Make sure to setup an Ingress or OpenShift route in the values file for external 
access of the UI (e.g. https://umbrella.domain.org). Also note that for the moment a dedicated API-Umbrella build needs to be used until 
the i4Trust related changes have been officially released: `fiware/api-umbrella:i4trust-rc3`. The issued private key and certificate 
chain must be added in PEM format. 
Make sure to use the chart from this [branch](https://github.com/FIWARE/helm-charts/tree/i4trust/charts/api-umbrella) until 
the chart has been officially released.
```shell
# Chart not officially released yet
#helm repo add fiware https://fiware.github.io/helm-charts/
#helm repo update
# Use https://github.com/FIWARE/helm-charts/tree/i4trust/charts/api-umbrella instead
helm install -f ./values/values-umbrella.yml --namespace provider api-umbrella fiware/api-umbrella --version 0.0.4
```

When first opening the page (https://umbrella.domain.org/admin), the credentials of the admin user can be set.

Within the Admin UI, create a new API Backend for the Orion Context Broker and configure it for the 
`Context Broker attribute based - iSHARE compliant` Authorization Mode, as depicted in the following pictures:

![API Umbrella API backend](./img/umbrella1.png "API Umbrella API Backend")

![API Umbrella authorization config](./img/umbrella2.png "API Umbrella API Backend authorization configuration")

API-Umbrella is now configured to receive requests at the `/packetdelivery` endpoint, check the access rights from the 
policies at the authorisation registries and, if access is granted, to forward the requests to the Orion Context Broker.



## Activation Service

The activation service is required to proxy requests from the i4Trust Marketplace for creating policies when companies 
acquired access to the data service. For this it provides endpoints `/token` and `/createpolicy` according to the iSHARE
scheme.

Modify the Activation Service [values file](./values/values-activation-service.yml) according to your needs and deploy 
the Activation Service. 
Make sure to setup an Ingress in the values file for external 
access (e.g. https://activation-service.domain.org). The issued private key and certificate 
chain must be added in PEM format.
```shell
helm repo add i4trust https://i4trust.github.io/helm-charts
helm repo update
helm install -f ./values/values-activation-service.yml --namespace provider activation-service i4trust/activation-service
```


## Packet Delivery Portal Demo Application

This a demo application of a packet delivery portal allowing external users to view and change their 
delivery orders stored at the Orion Context Broker. For accessing the service via API-Umbrella, the users and 
connected retailer companies need the required policies at the authorisation registries.

This demo is intended to showcase how to setup an application that access the provided data service via API-Umbrella.

Modify the PDC Portal [values file](./values/values-pdc-portal.yml) according to your needs and deploy 
the portal application. 
Make sure to setup an Ingress in the values file for external 
access (e.g. https://pdc-portal.domain.org). The issued private key and certificate 
chain must be added in PEM format.
```shell
helm repo add i4trust https://i4trust.github.io/helm-charts
helm repo update
helm install -f ./values/values-pdc-portal.yml --namespace provider pdc-portal i4trust/pdc-portal
```



