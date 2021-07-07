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

Is is assumed that all components will be deployed within the namespace `provider`. Change this name according to your 
needs.
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
helm install -f ./values/values-keyrock.yml --namespace provider keyrock fiware/keyrock --version 0.1.0
```

In a browser open the Keyrock UI (e.g. https://keyrock.domain.org) and login with the admin credentials provided in 
the values file. Then users can be created by the Admin user, or users sign up on their own.



## API-Umbrella

API-Umbrella is used as Policy Enforcement Point (PEP) and Policy Decision Point (PDP).

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
Check that in the database configuration, you provide the same password for the database user as has been used when creating 
the MongoDB database and user.
Make sure to setup an Ingress or OpenShift route in the values file for external 
access of the UI (e.g. https://umbrella.domain.org). The issued private key and certificate 
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
the Activation Service. Especially configure the authorisation Registry used.
Make sure to setup an Ingress in the values file for external 
access (e.g. https://activation-service.domain.org). The issued private key and certificate 
chain must be added in PEM format.
```shell
helm repo add i4trust https://i4trust.github.io/helm-charts
helm repo update
helm install -f ./values/values-activation-service.yml --namespace provider activation-service i4trust/activation-service
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
helm install -f ./values/values-pdc-portal.yml --namespace provider pdc-portal i4trust/pdc-portal
```


### Granting access for a consuming party

In order that users of an external organisation can access certain entities via the portal application, 
a policy at the provider's authorisation registry needs to be created first, which grants access to the organisation 
and allows the organisation to delegate the access to it's users. This builds a delegation chain so that the 
service provider does only need to know it has granted access to the specific organisation, but does not 
need to know anything about the user which finally accesses the service.

As an example, for an organisation Happy Pets with EORI `EU.` being granted read access (the portal performs GET requests 
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
