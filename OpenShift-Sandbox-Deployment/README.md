# A Minimal i4Trust deployment using the Red Hat OpenShift Sandbox

This is a simple _"Getting started with i4Trust"_ installation guide, targeting developers with limited Kubernetes experience and
therefore not already having a working cluster available on the cloud. We use the [Red Hat OpenShift Developer Sandbox](https://developers.redhat.com/developer-sandbox) 
to create a fully working OpenShift environment to build on, without any cost. The cluster itself has some restrictions and should not
be used for production purposes, but is very convenient to use as a starting point. Follow the steps described below, and all necessary
components will be available and up-and-running properly for development and testing.

> :warning: This is a simplified _"Getting started"_ set-up designed for tutorial use only, and should NEVER be used as a production environment.
> It is not properly secured as it contains plain-text passwords, and also contains other obvious anti-patterns, for example it has no resource
> limiting, no availability settings ans no backups. Copying the tutorial set-up directly for commercial use will harm you in production. This is 
> a just development setup, and should be treated as such.

## Background

[i4Trust](https://i4trust.org/) is a common framework used for creating collaborative dataspaces based on [FIWARE](https://www.fiware.org/)
and [iSHARE](https://dev.ishareworks.org/index.html). Data is passed between context brokers using the 
[NGSI-LD data](https://en.wikipedia.org/wiki/NGSI-LD) format, and protected using distributed trust mechanisms, meaning that a permission 
chain of _"who gives access to what"_ is set up between the  various participants within the dataspace. Individual clients of the particpants
do not need to be preregistered, but can be vouched for by an existing participant who has already registered a digital certificate created by a Certificate Authority.

What this means in practice is that, to allow its users to take part in a dataspace, a participant will need to create a publicly available
cloud based environment consisting of their own [context broker](https://github.com/FIWARE/context.Orion-LD) protected by a 
[PEP-Proxy](https://github.com/FIWARE/kong-plugins-fiware) which delegates appropriate security actions up to an [iSHARE satellite](https://ishare.eu/ishare-satellite-explained/). To create this we will need to use common cloud orchestration tools such as [Kubernetes](https://kubernetes.io/) and [Helm Charts](https://helm.sh/docs/topics/charts/).

## Prerequisites

This tutorial assumes some background in cloud operations, as you can find plenty of [existing tutorials](https://developers.redhat.com/developer-sandbox/activities/learn-kubernetes-using-red-hat-developer-sandbox-openshift) elsewhere on the Internet. Please ensure that you have downloaded the [oc](https://developers.redhat.com/articles/2022/06/16/learn-about-openshift-command-line-tools) , [helm](https://helm.sh/docs/intro/install/) and [kubectrl](https://kubernetes.io/docs/reference/kubectl/) command line clients and familiarised yourself with their use before starting the tutorial. Basic knowledge of cryptography such as how to create or manipulate [digital certificates](https://en.wikipedia.org/wiki/Public_key_certificate) using [openssl](https://www.openssl.org/) and [base 64 encoding](https://en.wikipedia.org/wiki/Base64) is also required.

### :movie_camera: Explainer Videos

The following explainer videos go into to more detail about the components and their usage

-  [iSHARE - Distributed Trust Basics](https://www.youtube.com/watch?v=_bvXbmvY8e8)
-  [i4Trust - how all of it fits together](https://www.youtube.com/watch?v=l5GoMuXBofQ)


## Scenario description

> :warning: The consumer "EU.EORI.NLHAPPYPETS" and the provider "EU.EORI.NLPACKETDEL" are placeholder and should be replaced with your valid participants certificates. Beside that, the ID of the provider("EU.EORI.NLPACKETDEL") also needs to be changed according to the used certificates in [kong-configuration](./kong/templates/configmap.yaml#l37) and [keyrock-configuration](./keyrock/values.yaml#l102).

The step-by-step guide will examplify an M2M-scenario, where consumer "EU.EORI.NLHAPPYPETS" requests the IDM(e.g. Keyrock) of provider "EU.EORI.NLPACKETDEL" for an access-token. This token is then used by "EU.EORI.NLHAPPYPETS" to request the orion-ld on "EU.EORI.NLPACKETDEL" through kong. Kong will verify the identity provided in the token, using the iShare-testinstances for satellite(e.g. https://scheme.isharetest.net) and authorization-registry(e.g. https://ar.isharetest.net). 

> :warning: If you don't have identities for the iShare-testinstances, either request them at the [i4Trust-helpdesk](https://spaces.fundingbox.com/c/i4trust/categories/i4TrustHelpDesk) or use a dedicated AR and Satellite(for example Keyrock and [FIWARE/ishare-satellite](https://github.com/FIWARE/ishare-satellite)).


## Step-by-step guide

1. Create account and sandbox on the [offical page from RedHat](https://developers.redhat.com/developer-sandbox/get-started)
launch 

![Launch sandbox](./doc/launch-sandbox.png)

2. Follow the guide to the openshift-console:
![Sandbox](./doc/sandbox.png)

3. Get the token and login from your console:

![Login-Link](./doc/login-link.png)

---

![Account](./doc/dev-sandbox.png)

---

![Token](./doc/token.png)

--> login using your local shell

4. Install [helm](https://helm.sh/docs/intro/install/) andl [kubectl](https://kubernetes.io/docs/tasks/tools/) locally via the offical documentation.

5. Clone tutorials repository
```shell
    git clone git@github.com:i4Trust/tutorials.git
```

6. Move to sandbox folder:
```shell
    cd tutorials/OpenShift-Sandbox-Deployment/
```

7. Install mongoDB:

>:warning: The sandbox allows installations to a dev namepsace. The namespace (hereafter referred to as `<NAMESPACE>` will have the name `<YOUR_RED_HAT_ACCOUNT>-dev`

```shell
    helm dependency build ./mongodb/
    helm install mongodb ./mongodb/ -n <NAMESPACE>
```

Verify:
```shell
    kubectl get pods -n <YOUR_ACCOUNT>-dev

    NAME                       READY   STATUS    RESTARTS   AGE
    mongodb-7d4b49b5f9-q54tm   1/1     Running   0          61s
```

8. Install mysql:

```shell
    helm dependency build ./mysql/
    helm install mysql ./mysql/ -n <NAMESPACE>
```

Verify:
```shell
    kubectl get pods -n <NAMESPACE>

    NAME                       READY   STATUS    RESTARTS   AGE
    mongodb-7d4b49b5f9-gr2cj   1/1     Running   0          83s
    mysql-664b9568bf-sh9k2     1/1     Running   0          39s
```

9. Install keyrock:

> :warning: replace cert and key in the [secrets.yaml](./keyrock/templates/secrets.yaml) with your data. They need to be base64 encoded.
> The certificate and key need to correspond with the ID configured at [./keyrock/values.yaml#l102](./keyrock/values.yaml#l102)
>
>  More information on how to obtain a test i4Trust digital certificate and how to encode it can be found [here](./doc/testcert.md)

```shell
    helm dependency build ./keyrock/
    helm install keyrock ./keyrock/ -n <NAMESPACE>
```

Verify:

```shell
    kubectl get pods -n <NAMESPACE>

    NAME                       READY   STATUS    RESTARTS   AGE
    keyrock-0                  1/1     Running   0          75s
    mongodb-7d4b49b5f9-gr2cj   1/1     Running   0          12m
    mysql-664b9568bf-sh9k2     1/1     Running   0          11m
```

Get the address:

```shell 
    kubectl get route keyrock -n <NAMESPACE>

    NAME      HOST/PORT                                                          PATH   SERVICES   PORT   TERMINATION   WILDCARD
    keyrock   keyrock-stefan-fiware-dev.apps.sandbox.x8i5.p1.openshiftapps.com          keyrock    8080                 None
```

Open in browser http://<URL> and login with ```User: my-admin@mail.org Password: admin```

11. Install Orion-LD:

```shell
    helm dependency build ./orion-ld/
    helm install orion-ld ./orion-ld/ -n <NAMESPACE>
```

Verify:

```shell
    kubectl get pods -n <NAMESPACE>

    NAME                         READY   STATUS    RESTARTS   AGE
    keyrock-0                    1/1     Running   0          3h3m
    mongodb-64949fb874-8fsh2     1/1     Running   0          3m41s
    mysql-664b9568bf-sh9k2       1/1     Running   0          3h14m
    orion-ld-58ddb4bcfb-rpx4x    1/1     Running   0          32s
```

12. Install Kong:

Insert certificate and key into the [secret-file](./kong/templates/secret.yaml). Make sure to base64-encode it. The ID at [config-map#l37](./kong/templates/configmap.yaml#l37) has to match the certificate and key.
Update the configuration for the backend service(usually the context broker) to be secured in the [config-map](./kong/templates/configmap.yaml). 

> :warning: Do not forget the ```--skip-crds```. The sandbox does not allow cluster-wide CRDs and we dont need them in our use-case.

```shell
    helm dependency build ./kong/
    helm install kong ./kong/ -n <NAMESPACE> --skip-crds
```

Verify:

```shell
    kubectl get pods -n <YOUR_ACCOUNT>-dev

    NAME                         READY   STATUS    RESTARTS   AGE
    keyrock-0                    1/1     Running   0          159m
    kong-kong-64fb477547-42d2p   1/1     Running   0          35s
    mongodb-7d4b49b5f9-96npm     1/1     Running   0          145m
    mysql-664b9568bf-sh9k2       1/1     Running   0          170m
```

The system now can be reached at: 

```shell
   kubectl get routes kong-route -n <NAMESPACE> -o json | jq -r '.spec.host'
```

## Try out the setup

Put your client key and certificate into the folder to mount(the files in [example-secrets](./doc/example-secrets/)) are only generated examples) and use the correct IDs.

1. Generate a [JWT](https://dev.ishareworks.org/introduction/jwt.html) for your client:
```shell
    docker run -v $(pwd)/doc/example-secrets:/certificates -e I_SHARE_CLIENT_ID="EU.EORI.NLHAPPYPETS" -e I_SHARE_IDP_ID="EU.EORI.NLPACKETDEL"  quay.io/wi_stefan/ishare-jwt-helper:0.2.1
```

The token will be print out on the command-line. Be aware that it is(as defined by iShare) only valid for 30s.

2. Request a token: 
```shell 
    curl --location --request POST 'https://<KEYROCK_URL/oauth2/token' \
        --header 'Content-Type: application/x-www-form-urlencoded' \
        --data-urlencode 'grant_type=urn:ietf:params:oauth:grant-type:jwt-bearer' \
        --data-urlencode 'scope=iSHARE' \
        --data-urlencode 'client_assertion_type=urn:ietf:params:oauth:client-assertion-type:jwt-bearer' \
        --data-urlencode 'client_assertion=<GENERATED_TOKEN>' \
        --data-urlencode 'client_id=EU.EORI.NLHAPPYPETS'
```

The response will look  like:

```json
{
    "access_token": "<TOKEN>",
    "expires_in": 3600,
    "token_type": "Bearer"
}
```

3. Request at kong:

```shell
curl --location --request GET 'http://<KONG-ADDRESS>/orion/ngsi-ld/v1/entities/urn:ngsi-ld:TEST:ENTITY' \
--header 'Authorization: Bearer <ACCESS_TOKEN>' 
```
    
This should lead to a response like:
    
```json
  {
      "message": "Local AR policy not authorized: Policy has expired or is not yet valid"
  }
```
The concrete output depends on the policies created for your participants(e.g. I_SHARE_CLIENT_ID and I_SHARE_IDP_ID). In our example, a policy is found but currently not active.
