# Data Service Provider

This describes how to setup an NGSI-LD based data service provider within the i4Trust trusted data space with the 
example of a fictitious Packet Delivery Company, which will provide digital packet delivery services.

The actual NGSI-LD data service is provided by an instance of the Orion-LD Context Broker. The following instructions 
describe how to deploy all required components on Kubernetes using Helm charts.

The whole environment will consist of the following components:
* [MongoDB](#mongodb)
* [MySQL](#mysql)
* [Orion Context Broker](#context-broker-orion)
* [Keyrock (Identity Provider)](#keyrock)
  - [Keyrock AR functionality](#keyrock-authorization-registry-functionality)
* [VC Verifier](#vc-verifier)
  - [VCWaltid](#vcwaltid)
  - [VCBackend](#vcbackend)
* [PEP Proxy/PDP](#pep-proxy--pdp)
  - [DSBA-PDP](#dsba-pdp)
  - [Kong](#kong)
  - [Deprecated: API-Umbrella + elasticsearch](#deprecated-api-umbrella)
* [Activation Service](#activation-service)
* [Portal demo application](#packet-delivery-portal-demo-application)
  - [Enable VerifiableCredentials](#enable-verifiablecredentials-for-the-portal)

as depicted in the following diagram:

![Components](./img/components.png "Components")

Furthermore it is required that there is access to an iSHARE Satellite instance and an iSHARE-compliant Authorisation 
Registry. It is assumed that the data service provider is already registered at the iSHARE Satellite and that 
certificates, a private key and the EORI have been issued. Starting with Keyrock Release 8.0.0, Keyrock provides it's own 
iSHARE-compliant Authorisation Registry and can be used instead.

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

The Kong helm chart with all possible configuration values can be found here:
* [Kong](https://github.com/Kong/charts)

Is is assumed that all components will be deployed within the namespace `provider`. Change this name according to your 
needs.
```shell
kubectl create ns provider
```

Due to the iSHARE specification, requests can contain very large headers with the signed JWTs. 
When using Kubernetes, note that the ingress controller must be capable of handling large request headers. When using 
nginx as ingress controller, these are the proposed parameters to be set:
```yaml
large-client-header-buffers: "8 32k"
http2-max-field-size: "32k"
http2-max-header-size: "32k"
```

## MongoDB

First modify the [Helm values file](./values/values-mongodb.yml) according to your environment and 
then deploy `mongodb`:
```shell
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm install -f ./values/values-mongodb.yml --namespace provider mongodb bitnami/mongodb --version 10.30.12
```

## MySQL

First modify the [values file](./values/values-mysql.yml) according to your needs and then deploy the MySQL database using `helm`. 
```shell
helm repo add t3n https://storage.googleapis.com/t3n-helm-charts
helm repo update
helm install -f ./values/values-mysql.yml --namespace provider mysql t3n/mysql --version 0.1.0
```

## Context Broker orion

First modify the [Helm values file](./values/values-orion.yml) according to your environment and 
then deploy `orion`:
```shell
helm repo add fiware https://fiware.github.io/helm-charts/
helm repo update
helm install -f ./values/values-orion.yml --namespace provider orion fiware/orion
```

### Creating entites for delivery orders

In order to follow the example of the Packet Delivery Company, there must be pre-existing delivery orders at the 
Packet Delivery Company. This means, that entities of type `DELIVERYORDER` need to be created at the Context Broker 
before. Below is an example of the body for the request when creating such an entity at Orion:
```json
{
    "id": "urn:ngsi-ld:DELIVERYORDER:HAPPYPETS001",
    "type": "DELIVERYORDER",
    "issuer": {
        "type": "Property",
        "value": "Happy Pets"
    },
    "destinee": {
        "type": "Property",
        "value": "Happy Pets customer"
    },
    "deliveryAddress": {
        "type": "Property",
        "value": {
            "addressCountry": "DE",
            "addressRegion": "Berlin",
            "addressLocality": "Berlin",
            "postalCode": "12345",
            "streetAddress": "Customer Strasse 23"
        }
    },
    "originAddress": {
        "type": "Property",
        "value": {
            "addressCountry": "DE",
            "addressRegion": "Berlin",
            "addressLocality": "Berlin",
            "postalCode": "12345",
            "streetAddress": "HappyPets Strasse 15"
        }
    },
    "pda": {
        "type": "Property",
        "value": "2021-10-03"
    },
    "pta": {
        "type": "Property",
        "value": "14:00:00"
    },
    "eda": {
        "type": "Property",
        "value": "2021-10-02"
    },
    "eta": {
        "type": "Property",
        "value": "14:00:00"
    },
    "@context": [
        "https://schema.lab.fiware.org/ld/context",
        "https://uri.etsi.org/ngsi-ld/v1/ngsi-ld-core-context.jsonld"
    ]
}
```
Feel free to create several entities with different properties and entity IDs. Later on these will be accessible 
by the user via the portal application.

## Keyrock

The Keyrock Identity Provider is required for storing the user accounts of the employees of the data service provider's 
company. In the experimentation framework example, it is needed in order that employees can login at the i4Trust 
Marketplace and create offerings on behalf of their company.

Modify the Keyrock [values file](./values/values-keyrock.yml) according to your needs and deploy the Keyrock Identity Provider. 
When there is no external authorisation registry configured for Keyrock, it will use it's internal authorisation registry and 
policies need to be stored there.
Make sure to setup an Ingress or OpenShift route in the values file for external 
access of the UI (e.g. https://keyrock.domain.org). Also note that for the moment a dedicated Keyrock build needs to be used until 
the i4Trust related changes have been officially released. The issued private key and certificate 
chain must be added in PEM format. 
```shell
helm repo add fiware https://fiware.github.io/helm-charts/
helm repo update
helm install -f ./values/values-keyrock.yml --namespace provider keyrock fiware/keyrock --version 0.4.6
```

In a browser open the Keyrock UI (e.g. https://keyrock.domain.org) and login with the admin credentials provided in 
the values file. Then users can be created by the Admin user, or users sign up on their own.

When using the internal authorisation registry of Keyrock, one might need to increase the maximum header size of the 
internal web server by setting the ENV, e.g. to
```shell
IDM_SERVER_MAX_HEADER_SIZE=32786
```
See the [values file](./values/values-keyrock.yml) for an example.

### Keyrock Authorization Registry functionality

Keyrock also allows to serve as Authorization Registry (AR) in order to store access policies in its own database. 
Keyrock then implements the [iSHARE /delegation endpoint](https://dev.ishareworks.org/delegation/endpoint.html) 
which allow to query for delegation evidences. 

In order to activate the AR functionality, the following parameter needs to be set in 
the [values file](./values/values-keyrock.yml):
```yaml
authorisationRegistry:
  url: "internal"
```
In this case, the parameters for `tokenEndpoint` and `delegationEndpoint` are ignored. 
The parameter for the AR `identifier` needs to be changed to the EORI of the organisation 
that operates this Keyrock/AR instance.

When activated, Keyrock provides the following endpoints:
* `/ar/delegation`: Query/obtain delegation evidences
* `/ar/policy`: Create delegation evidences

Access tokens for the AR need to be retrieved from the `/oauth2/token` endpoint, as described 
in the [iSHARE Access Token specification](https://dev.ishareworks.org/common/token.html).

## VC Verifier

In order to use [VerifiableCredentials](https://www.w3.org/TR/vc-data-model/) for authentication and authorization, 2 additional components need to be deployed: 
- the [VCBackend](https://github.com/FIWARE/VCBackend) that can server as Verifier and Issuer
- the [VCWaltid](https://github.com/FIWARE/VCWaltid) to provide the VerifiableCredential Type(```PacketDeliveryService```) that is used in the tutorial

### VCWaltid

Since the VCBackend uses the VCWaltid, an instance needs to be deployed. The provided instance of VCWaltid supports the creation of credentials of the following format:

```json
{
  "@context": [
    "https://www.w3.org/2018/credentials/v1",
    "https://w3id.org/security/suites/jws-2020/v1"
  ],
  "credentialSchema": {
    "id": "https://raw.githubusercontent.com/hesusruiz/dsbamvf/main/schemas/PacketDeliveryService/2022-10/schema.json",
    "type": "FullJsonSchemaValidator2021"
  },
  "credentialSubject": {
    "familyName": "Customer",
    "firstName": "Happy",
    "id": "did:key:z6Mkfdio1n9SKoZUtKdr9GTCZsRPbwHN8f7rbJghJRGdCt88",
    "roles": [
      {
        "names": [
          "GOLD_CUSTOMER"
        ],
        "target": "EU.EORI.NLPACKETDEL"
      }
    ]
  },
  "id": "urn:uuid:b84ceee3-7cc6-4a1c-b0c2-4fbffe8346d2",
  "issuanceDate": "2022-12-13T14:09:43Z",
  "issued": "2022-12-13T14:09:43Z",
  "issuer": "did:key:z6MkufEFkFSCXg1DHUsZbYuWcGf4QHat34YSoVgs67ED9m4S",
  "proof": {
    "created": "2022-12-13T14:09:44Z",
    "creator": "did:key:z6MkufEFkFSCXg1DHUsZbYuWcGf4QHat34YSoVgs67ED9m4S",
    "jws": "eyJiNjQiOmZhbHNlLCJjcml0IjpbImI2NCJdLCJhbGciOiJFZERTQSJ9..vMs2iaQddhKDhu4tQMdHxqvMu9ksdxiYr2Q-dXXSRoDEmZyyfw5VsoPXJmVRBauovpFXatL9Je7zjSu9zSo7AA",
    "type": "JsonWebSignature2020",
    "verificationMethod": "did:key:z6MkufEFkFSCXg1DHUsZbYuWcGf4QHat34YSoVgs67ED9m4S#z6MkufEFkFSCXg1DHUsZbYuWcGf4QHat34YSoVgs67ED9m4S"
  },
  "type": [
    "VerifiableCredential",
    "PacketDeliveryService"
  ],
  "validFrom": "2022-12-13T14:09:43Z"
}
```
Install VCWaltid via:

```shell
helm repo add i4trust https://i4trust.github.io/helm-charts
helm repo update
helm install -f ./values/values-verifier-waltid.yml --namespace provider waltid i4trust/vcwaltid --version 0.0.4
```
VCWaltid is now deployed and can be used.

### VCBackend

To verify the credentials, VCBackend's Verifier has to be provided. The verifier endpoint needs to be public available to execute the SIOP flow. Depending on your cluster, configure either the ingress(for most k8s) or the route(for OpenShift). Check the documentation of the chart: https://github.com/i4Trust/helm-charts/blob/main/charts/vcbackend/values.yaml#L92-L129 for more.

The verifier than can be installed via:

```shell
helm repo add i4trust https://i4trust.github.io/helm-charts
helm repo update
helm install -f ./values/values-verifier-backend.yml --namespace provider waltid i4trust/vcbackend --version 0.0.8
```

Once it started, check the log for the DID's of the backend, they will be needed for later usage. The log will look like the following:

```
2022-12-21T06:21:28.199Z	INFO	app/main.go:144	SSIKit is configured at: &{http://i4trust-demo-pdc-waltid-vcwaltid:7000 http://i4trust-demo-pdc-waltid-vcwaltid:7001 http://i4trust-demo-pdc-waltid-vcwaltid:7002 http://i4trust-demo-pdc-waltid-vcwaltid:7003 http://i4trust-demo-pdc-waltid-vcwaltid:7010}
2022-12-21T06:21:28.199Z	INFO	app/main.go:151	IssuerDID created	{"did": "did:key:z6MkufEFkFSCXg1DHUsZbYuWcGf4QHat34YSoVgs67ED9m4S"}
2022-12-21T06:21:28.199Z	INFO	app/main.go:157	VerifierDID created	{"did": "did:key:z6Mkk5iPrXg35fC4aq4yp3QadqVGKFhQL2b76fy6QKmSXJNT"}	
``` 

## PEP Proxy / PDP

### DSBA-PDP

In order to allow authorization with VerifiableCredentials, using the SIOP2 flow, a PDP evaluating the VC and the corresponding policies is required. The PDP will provide an `/authz` endpoint to receive a JWT containing the (already verified) VerifiableCredential. Beside the token, it expects the original request address(`X-Original-URI`), method (`X-Original-Action`) and request body as body. 

The [DSBA-PDP](https://github.com/wistefan/dsba-pdp) can be installed, using its [HELM-Chart](https://github.com/FIWARE/helm-charts/tree/main/charts/dsba-pdp) and the provided [values-file](./values/values-dsba-pdp.yml). For detailed documentation on the configuration options, check the documentation of the original chart. In order to allow secret provisioning of database passwords and iShare-credentials, two k8s-secrets are provided: 
* [pdp-db-secret](./secret/pdp-db-secret.yml) - contains db password(s) as expected by the recommended DB chart and the DSBA-PDP
* [pdp-ishare-secret](./secret/pdp-ishare-secret.yml) - contains the certificate and key of the iShare-participant(e.g. Packetdelivery)  

> :bulb: To create a valid ishare-secret form a given certificate and key, the following command can be used: `kubectl create secret generic pdp-ishare-secret --dry-run --from-file certificate.pem --from-file key.pem -o yaml > pdp-ishare-secret.yml`. The PDP expects the secret to contain an `certificate.pem` and a `key.pem` entry, thus rename either the files or the entries inside secret to match that expectation.

When the secrets are properly prepared, install them via:

```shell
kubectl apply -f ./secret/pdp-db-secret.yml -n provider
kubectl apply -f ./secret/pdp-ishare-secret.yml -n provider
```

In order to provide persistence for the trustedissuers-list, a database is required. It can be installed via:

```shell
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo updatea
helm install -f ./values/values-mysql-pdp.yml --namespace provider mysql-pdp bitnmi/mysql --version 9.4.4
```
Change the [values-file](./values/values-mysql-pdp.yml) according to your needs. Check the [chart-documentation](https://github.com/bitnami/charts/tree/main/bitnami/mysql) for all options.

The DSBA-PDP can then be installed using the provided [values-file](./values/values-dsba-pdp.yml), too:

```shell
helm repo add fiware https://fiware.github.io/helm-charts/
helm repo update
helm install -f ./values/values-dsba-pdp.yml --namespace provider dsba-pdp fiware/dsba-pdp --version 0.0.11
```

The PDP is now deployed and can be used.

### Kong

Kong is an API Gateway. With the usage of the 
[ngsi-ishare-policies](https://github.com/FIWARE/kong-plugins-fiware/tree/main/kong-plugin-ngsi-ishare-policies) 
plugin, it can be used as Policy Enforcement Point (PEP) and Policy Decision Point (PDP).

Modify the [Kong values file](./values/values-kong.yml) and 
[Kong declarative setup configuration file](./kong/kong.yml). Then perform the deployment using 
Helm.  
Depending on whether you use an external or the Keyrock built-in authorisation registry, it's endpoints and configuration 
parameters need to be configured accordingly.
Make sure to setup an Ingress or OpenShift route in the values file for external 
access to the provided Kong endpoints/routes. The issued private key and certificate 
chain must be added in PEM format. 

```shell
helm repo add kong https://charts.konghq.com
helm repo update
helm install -f ./values/values-kong.yml --namespace provider kong kong/kong --set ingressController.installCRDs=false --set-file dblessConfig.config=./kong/kong.yml --version 2.8.0
```

Kong is now configured to receive requests at the `/packetdelivery` endpoint, check the access rights from the 
policies at the authorisation registries and, if access is granted, to forward the requests to the Orion Context Broker.
Beside the `/packetdelivery` endpoint, Kong is also configured to support the usage of verifiable credentials. When using an JWT containing a VerifiableCredential, use the endpoint `/packetdelivery-vc`. For the endpoint to work, the [DSBA-PDP](https://github.com/wistefan/dsba-pdp) has to be installed first.

### Deprecated: API-Umbrella

API-Umbrella can be used as Policy Enforcement Point (PEP) and Policy Decision Point (PDP) as well.

It requires an instance of elasticsearch:  
First modify the [Helm values file](./values/values-elastic.yml) according to your environment and 
then deploy `elasticsearch`:
```shell
helm repo add elastic https://helm.elastic.co
helm repo update
helm install -f ./values/values-elastic.yml --namespace provider elasticsearch elastic/elasticsearch --version 7.5.1
```

During deployment of the MongoDB using the provided values file, a database should have been created for API Umbrella. 
In case this was removed from the values file, it is needed to create the necessary database and user within the MongoDB 
manually. This requires to enter a shell within the MongoDB pod, 
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
the deployment using Helm.  
Note, for the verification of signed JWTs according to iSHARE specifications, you either need to configure the iSHARE Satellite 
endpoint or provide the root CA.  
Check that in the database configuration, you provide the same password for the database user as has been used when creating 
the MongoDB database and user.  
Depending on whether you use an external or the Keyrock built-in authorisation registry, it's endpoints and configuration 
parameters need to be configured accordingly.
Make sure to setup an Ingress or OpenShift route in the values file for external 
access of the UI (e.g. https://umbrella.domain.org). The issued private key and certificate 
chain must be added in PEM format. 
```shell
helm repo add fiware https://fiware.github.io/helm-charts/
helm repo update
helm install -f ./values/values-umbrella.yml --namespace provider api-umbrella fiware/api-umbrella --version 0.0.10
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
the Activation Service. Especially configure the authorisation Registry used.
Make sure to setup an Ingress in the values file for external 
access (e.g. https://activation-service.domain.org). The issued private key and certificate 
chain must be added in PEM format.
```shell
helm repo add i4trust https://i4trust.github.io/helm-charts
helm repo update
helm install -f ./values/values-activation-service.yml --namespace provider activation-service i4trust/activation-service --version 1.1.0
```

In order to allow external parties to create policies on behalf of the Packet Delivery Company, a policy needs to be created 
for these external organisations. This would be the case, when the data service would be offered on a marketplace.
As an example, considering a marketplace with EORI `EU.EORI.NLMARKETPLA` should be allowed 
to create policies when organisations acquire access to the service offering, the policy to be created would look 
like this:
```json
{
	"delegationEvidence": {
		"notBefore": 1614354348,
		"notOnOrAfter": 1654083306,
		"policyIssuer": "EU.EORI.NLPACKETDEL",
		"target": {
			"accessSubject": "EU.EORI.NLMARKETPLA"
		},
		"policySets": [
			{
				"policies": [
					{
						"target": {
							"resource": {
								"type": "delegationEvidence",
								"identifiers": [
									"*"
								],
								"attributes": [
									"*"
								]
							},
							"actions": [
								"POST"
							]
						},
						"rules": [
							{
								"effect": "Permit"
							}
						]
					}
				]
			}
		]
	}
}
```
Make sure to adapt the expiration timestamp accordingly.



## Packet Delivery Portal Demo Application

This is a demo application of a packet delivery portal allowing external users to view and change their 
delivery orders stored at the Orion Context Broker. For accessing the service via API-Umbrella, the users and 
connected retailer companies need the required policies at the authorisation registries.

This demo is intended to showcase how to setup an application that access the provided data service via API-Umbrella.

Modify the PDC Portal [values file](./values/values-pdc-portal.yml) according to your needs. 

In the portal application, there are two static login options implemented, targeting at two different 
IDPs (Keyrock) of consuming organisations (e.g., Happy Pets and No Cheaper) that need to have access to the service 
offering. In the values file, 
adapt the IDP endpoints according to your setup. Also check the instructions about a 
[Data Service Consumer](../Data-Service-Consumer). 

Make sure to setup an Ingress in the values file for external 
access (e.g. https://pdc-portal.domain.org). The issued private key and certificate 
chain must be added in PEM format.

The portal application can then be deployed using:
```shell
helm repo add i4trust https://i4trust.github.io/helm-charts
helm repo update
helm install -f ./values/values-pdc-portal.yml --namespace provider pdc-portal i4trust/pdc-portal --version 2.1.0
```

### Enable VerifiableCredentials for the Portal

In order to setup the portal-application for using VerifiableCredentials, the following parts of the configuration needs to be set:

```json
config:

	...

	# Context Broker configuration
	cb:
		# Endpoint of (Kong protected) NGSI-LD API for i4Trust Standard flow
		endpoint: "https://kong.domain.org/packetdelivery/ngsi-ld/v1"
		# Endpoint of (Kong protected) NGSI-LD API for the Verifiable Credentials - needs to be the path configured in the kong.yml
		endpoint_siop: "https://kong.domain.org/orion-vc/ngsi-ld/v1"

	# Configuration for SIOP flow
	siop:
		# SIOP flow enabled
		enabled: true
		# Redirect URI that the wallet will use to send the VC/VP - this is the host configured by the ingress/route of the vcbackend-component
		redirect_uri: https://vcbackend.domain.org/verifier/api/v1/authenticationresponse
		# Base uri of the verifier - this is the host configured by the ingress/route of the vcbackend-component
		verifier_uri: https://vcbackend.domain.org
		# DID of verifier - the did is printed out to the log on startup of the verifier (see the section on installing [VCBackend](#vcbackend))
		did: "did:key:<VERIFIER_KEY>"
		# Type of credential that the Verifier will accept
		scope: "dsba.credentials.presentation.PacketDeliveryService"
```

### Granting access for a consuming party

In order that users of an external organisation can access certain entities via the portal application, 
a policy at the provider's authorisation registry needs to be created first, which grants access to the organisation 
and allows the organisation to delegate the access to it's users. This builds a delegation chain so that the 
service provider does only need to know it has granted access to the specific organisation, but does not 
need to know anything about the user which finally accesses the service.

As an example, for an organisation Happy Pets with EORI `EU.EORI.NLHAPPYPETS` being granted read access (the portal performs GET requests 
when accessing a certain entity) to all entities of type 
`DELIVERYORDER` for all attributes, and write access (the portal performs PATCH requests when updating certain 
attributes) for the attributes of `pta` and `pta`, such policy would look like the following:
```json
{
	"delegationEvidence": {
		"notBefore": 1624634606,
		"notOnOrAfter": 1624636406,
		"policyIssuer": "EU.EORI.NLPACKETDEL",
		"target": {
			"accessSubject": "EU.EORI.NLHAPPYPETS"
		},
		"policySets": [
			{
				"maxDelegationDepth": 0,
				"target": {
					"environment": {
						"licenses": [
							"ISHARE.0001"
						]
					}
				},
				"policies": [
					{
						"target": {
							"resource": {
								"type": "DELIVERYORDER",
								"identifiers": [
									"*"
								],
								"attributes": [
									"pta",
									"pda"
								]
							},
							"actions": [
								"PATCH"
							]
						},
						"rules": [
							{
								"effect": "Permit"
							}
						]
					},
					{
						"target": {
							"resource": {
								"type": "DELIVERYORDER",
								"identifiers": [
									"*"
								],
								"attributes": [
									"*"
								]
							},
							"actions": [
								"GET"
							]
						},
						"rules": [
							{
								"effect": "Permit"
							}
						]
					}
				]
			}
		]
	}
}
```
Note that such policy would be created by the marketplace, in the case that the offering is acquired by 
a certain organisation like Happy Pets.



### Granting access to a certain user

In the environment of the consuming organisation, more precisely that means the organisation that was granted access to 
the service as described in the previous section, access policies need to be delegated to it's users that 
should be allowed to access the service of the provider.

For this, check the instructions about the [Data Service Consumer](../Data-Service-Consumer).

When logging in at the portal application via the consuming organisation's (e.g. Happy Pets) Keyrock IDP, 
the user receives an iSHARE-compliant JWT which will be sent along all requests to the provider's packet delivery service 
endpoint, namely the API Umbrella instance performing the access management for the service provider's 
context broker.

Depending on whether the consuming organisation's Keyrock instance was configured using it's own authorisation registry 
or an external one, the JWT will contain either the user's policies directly or access information about the external 
authorisation registry, which will allow API Umbrella to check for the necessary access rights of the user.
