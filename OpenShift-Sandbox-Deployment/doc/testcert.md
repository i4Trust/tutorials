# Creating and Manipulating test Digital Certificates for i4Trust

You create can i4Trust test digital certificates in the following resource: `https://ca7.isharetest.net:8442/ejbca/ra/index.xhtml` 

Click on **Make New Request**, then on **postpone**. You will be redirected to an interface, where you need to fill-in organization details. 
You decide on the organization "test" EORI in the textbox called "serialNumber (in DN)". 

The resultant p12 file contains a digital certificate that uses [PKCS#12](https://en.wikipedia.org/wiki/PKCS_12) 
(Public Key Cryptography Standard #12) encryption. It is used as a portable format for transferring personal private keys and 
other sensitive information.


Use [openssl](https://en.wikipedia.org/wiki/OpenSSL) to extract a certificate chain from a P12 file:

```console
openssl pkcs12 -info -in <filename>.p12 -out <filename>.certs -nodes -nokeys
```

To extract a key from a P12 file:

```console
openssl pkcs12 -info -in <filename>.p12 -out <filename>.key -nodes -nocerts
```

To switch a key to RSA format:

```console
openssl rsa -in <filename>.key -out <filename>.rsa
```

###  Switching to BASE64

Note that an extracted certificate chain will also include Certificate bag information which is unnecessary 
for i4Trust components:

```text
Certificate bag
Bag Attributes
    localKeyID: 81 52 E0 F5 EF 78 9E 10 98 55 79 14 EC 3D F4 FB 46 EA AD EF 
    friendlyName: PacketDeliveryCo
subject=/CN=PacketDeliveryCo/serialNumber=EU.EORI.NLPACKETDEL/O=Packet Delivery Co/C=NL
issuer=/CN=iSHARETestCA_TLS/OU=Test/O=iSHARE/C=NL
-----BEGIN CERTIFICATE-----
XXXXX==
-----END CERTIFICATE-----
Certificate bag
Bag Attributes
    friendlyName: iSHARETestCA_TLS
subject=/CN=iSHARETestCA_TLS/OU=Test/O=iSHARE/C=NL
issuer=/CN=iSHARETestCA/OU=Test/O=iSHARE/C=NL
-----BEGIN CERTIFICATE-----
XXXXX==
-----END CERTIFICATE-----
Certificate bag
Bag Attributes
    friendlyName: iSHARETestCA
subject=/CN=iSHARETestCA/OU=Test/O=iSHARE/C=NL
issuer=/CN=iSHARETestCA/OU=Test/O=iSHARE/C=NL
-----BEGIN CERTIFICATE-----
XXXXX==
-----END CERTIFICATE-----
```

Before proceeding, this file needs to be manually edited as shown:

```text
-----BEGIN CERTIFICATE-----
XXXXX==
-----END CERTIFICATE-----
-----BEGIN CERTIFICATE-----
XXXXX==
-----END CERTIFICATE-----
-----BEGIN CERTIFICATE-----
XXXXX==
-----END CERTIFICATE-----
```


To convert a certificate chain to [base64](https://en.wikipedia.org/wiki/Base64) format: 

```console
base64 -i <filename>.certs -o <filename>.certs64
```

To convert a keyfile to base64 format: 

```console
base64 -i <filename>.key -o <filename>.key64
```
