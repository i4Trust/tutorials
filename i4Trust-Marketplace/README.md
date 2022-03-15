# Installing the i4Trust Marketplace on Kubernetes with Helm

The following describes how to setup a full instance of the FIWARE Business API Ecosystem (BAE) in the 
context of an i4Trust Data Space. This includes the 
BAE itself, as well as the required databases and an Identity Provider (Keyrock) for administrative 
access to the BAE.

This repository provides examples of the [Helm values files](./values) which show the minimum configuration 
parameters to be set. Adapt these for your setup before proceeding with the instructions.

The helm chart with all possible configuration values can be found here:
* [Sources](https://github.com/FIWARE/helm-charts/tree/main/charts/business-api-ecosystem)
* [Artifact HUB](https://artifacthub.io/packages/helm/fiware/business-api-ecosystem)


The setup within an i4Trust data space is based on the description 
in the [production-on-k8s](https://github.com/FIWARE/production-on-k8s/tree/main/business-api-ecosystem) repository 
of the [FIWARE Foundation](https://www.fiware.org). In the following only the differences to the linked setup will 
be presented.


## Preparation

Prepare Helm repositories
```shell
helm repo add t3n https://storage.googleapis.com/t3n-helm-charts
helm repo add elastic https://helm.elastic.co
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo add fiware https://fiware.github.io/helm-charts/
helm repo update
```

We will assume that all components will be deployed within the namespace `marketplace`.
```shell
kubectl create ns marketplace
```

Due to the iSHARE specification, requests can contain very large headers with the signed JWTs. 
When using Kubernetes, note that the ingress controller must be capable of handling large request headers. When using 
nginx as ingress controller, these are the proposed parameters to be set:
```yaml
large-client-header-buffers: "8 32k"
http2-max-field-size: "32k"
http2-max-header-size: "32k"
```

In the following we assume that you have control of the domain `domain.org` and that the EORI issued to the marketplace 
is `EU.EORI.NLMARKETPLA`. Furthermore we assume 
that the externally available components will be accessible on the following subdomains:
* Keyrock IdM: https://keyrock.domain.org
* BAE UI (Logic Proxy): https://marketplace.domain.org


## Databases

For the components of 
* MongoDB
* MySQL
* elasticsearch

there is no changed configuration compared to the instructions of the repository 
[production-on-k8s](https://github.com/FIWARE/production-on-k8s/tree/main/business-api-ecosystem). Just follow the steps 
described in the linked repository.


## Identity Provider (Keyrock)

An instance of the Keyrock Identity Provider dedicated to the BAE is required in order to have 
administrative access to the BAE. Note that the login via this Keyrock instance is performed based 
on the OAuth2 protocol, whereas the Identity Providers deployed at the environments of service providers and 
service consumers follow the OpenID Connect protocol based on iSHARE specifications.

Modify the Keyrock [values file](./values/values-keyrock.yml) according to your needs and deploy the Keyrock Identity Provider. 
Make sure to setup an Ingress or OpenShift route in the values file for external 
access of the UI (e.g. https://keyrock.domain.org). Also note that for the moment a dedicated Keyrock build needs to be used until 
the i4Trust related changes have been officially released: `fiware/idm:i4trust-rc4`. The issued private key and certificate 
chain must be added in PEM format. 
```shell
helm install -f ./values/values-keyrock.yml --namespace marketplace keyrock fiware/keyrock --version 0.2.3
```

In a browser open the Keyrock UI (e.g. https://keyrock.domain.org) and login with the admin credentials provided in 
the values file. Then follow the steps described for the Keyrock instance in 
[production-on-k8s](https://github.com/FIWARE/production-on-k8s/tree/main/business-api-ecosystem).

Users can be created by the Admin user, or users sign up on their own, and need to be assigned the roles 
described in [production-on-k8s](https://github.com/FIWARE/production-on-k8s/tree/main/business-api-ecosystem). Since this 
Keyrock instance is dedicated for having administrative access to the BAE, users basically only need to the `admin` role.



## Business API Ecosystem (Marketplace)

Finally install the Business API Ecosystem. Make sure to setup an Ingress or OpenShift route in the 
[values file](./values/values-marketplace.yml) for external 
access of the Marketplace UI (e.g. https://marketplace.domain.org). Furthermore adapt the configuration options for 
the databases, elasticsearch and Keyrock instances which have been setup before.

This values file incorporates the usage of the [i4Trust theme](https://github.com/i4Trust/bae-i4trust-theme) for the marketplace UI. 
External IDPs of service providers and service consumers, which are supposed to login to the marketplace via their own IDPs, 
need to be added via the Administration UI of the BAE after logging in with the `admin` role, in order to be selectable on the 
login dialog of the marketplace UI. In order to login using the dedicated Keyrock instance setup above, one needs to open 
this link: [https://marketplace.domain.org/login](https://marketplace.domain.org/login).

The private key and certificate chain issued for the marketplace must be added in PEM format. 
```shell
# Deploy BAE
helm install -f ./values/values-marketplace.yml --namespace marketplace bae fiware/business-api-ecosystem --version 0.2.6
```

The deployment of all components will take some time. When the logic proxy component has been deployed and changed to the running state, 
you can access the marketplace UI via the browser (e.g. http://marketplace.domain.org).

For logging in into the marketplace for administrative access via the Keyrock instance dedicated to the BAE, open this 
URL: [http://marketplace.domain.org/login](http://marketplace.domain.org/login). 

In order to support the asset type of an i4Trust-compliant NGSI-LD data service, a dedicated plugin needs to be added to the 
charging backend component. The plugin itself can be found [here](https://github.com/i4Trust/bae-i4trust-service), installation 
instructions can be found [here](https://business-api-ecosystem.readthedocs.io/en/latest/plugins-guide.html#installing-asset-plugins). 
On Kubernetes, basically one first needs to copy the zipped plugin to the `/plugins` directory of the charging backend pod and then 
perform the step from above instructions. Note that one needs to enable the plugins PVC for the charging backend in the Helm 
configuration, in order that plugins remain installed after the pod restarts.



## Additional information

### Auth scheme of MongoDB

Follow the instructions at 
[production-on-k8s](https://github.com/FIWARE/production-on-k8s/tree/main/business-api-ecosystem).

