# i4Trust Tutorials

[![License: MIT](https://img.shields.io/github/license/i4Trust/tutorials.svg)](https://opensource.org/licenses/MIT)
![](https://nexus.lab.fiware.org/repository/raw/public/badges/chapters/fiware-ops.svg)

Tutorials and descriptions on how to setup the components of an i4Trust data space.

[i4Trust](https://i4trust.org/) is a common framework used for creating collaborative dataspaces based on [FIWARE](https://www.fiware.org/)
and [iSHARE](https://dev.ishareworks.org/index.html). Data is passed between context brokers using the 
[NGSI-LD data](https://en.wikipedia.org/wiki/NGSI-LD) format, and protected using distributed trust mechanisms, meaning that a permission 
chain of _"who gives access to what"_ is set up between the  various participants within the dataspace. Individual clients of the particpants
do not need to be preregistered, but can be vouched for by an existing participant who has already registered a digital certificate created by a Certificate Authority.

## List of Tutorials

### 1. Red Hat OpenShift Sandbox

The [OpenShift-Sandbox-Deployment](./OpenShift-Sandbox-Deployment) provides an installation guide 
deploying a minimal example of i4Trust Building Blocks to the Red Hat OpenShift Sandbox.

### 2. Packet Delivery Reference Example

The [PacketDelivery-ReferenceExample](./PacketDelivery-ReferenceExample) provides an installation guide, 
to deploy the full environment of the i4Trust reference example use case of the Packet Delivery 
Company to a Kubernetes Cluster using Helm charts.

> :bulb: The support of VerfifiableCredentials for authorization via OIDC4VP/SIOP-2(as explained in the [training-material - p32-39](https://raw.githubusercontent.com/i4Trust/training/main/2_Technology/7_Specific_components_for_Data_Spaces/i4Trust%20Data%20Spaces%20-%20Detailed%20look%20into%20the%20reference%20example.pdf)) is currently under development and will be fully available by the end of January 2022. However, a working tutorial is already availble at the [add-vc branch](https://github.com/i4Trust/tutorials/tree/add-vc). Feel free to try it out already and make yourself familiar with new opportunities. 

---

## License

[MIT](LICENSE) © 2022-2023 FIWARE Foundation e.V.
