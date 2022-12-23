# Reference Example: Packet Delivery Use Case

This provides an installation guide, 
to deploy the full environment of the i4Trust reference example use case of the Packet Delivery 
Company to a Kubernetes Cluster using Helm charts.

A full description of this reference use case can be found in the i4Trust Building Blocks 
Document which can be downloaded [here](https://github.com/i4Trust/building-blocks).

The installation guide is separated into the three different parties/organisations of the data space. These should be treated 
as separate environments within the decentralised data space.

* [i4Trust-Marketplace](./i4Trust-Marketplace): The Marketplace instance
* [Data-Service-Provider](./Data-Service-Provider): The data service provider, e.g., Packet Delivery Company
* [Data-Service-Consumer](./Data-Service-Consumer): The data service consumer, e.g., Happy Pets

Note that when deploying the environment of No Cheaper as well, a separate Data Service Consumer environment 
needs to be deployed.

Once you set up the system according to the tutorial, checkout the [Bringing the pieces together](https://github.com/i4Trust/training/blob/main/2_Technology/7_Specific_components_for_Data_Spaces/i4Trust%20Data%20Spaces%20-%20Detailed%20look%20into%20the%20reference%20example.pdf) presentation, to find out about the capabilites and learn f.e. how to sell, serve and access services via VerifiableCredentials on the Marketplace.


