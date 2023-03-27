###

## Components

- Keycloak with postgres & config & VC-Issuer Plugin
- Data Consumer application? Happy Pets?
- WaltID needed for issuer only too?




  - name: vcwaltid
    version: 0.0.13
    repository: https://i4Trust.github.io/helm-charts


    helm install -ntim -f waltid/values.yaml waltid --repo https://i4Trust.github.io/helm-charts/ --version 0.0.13 vcwaltid --dry-run


    WaltID Get templates:
    curl -X 'GET' \
  'localhost:7001/v1/templates' \
  -H 'accept: application/json'



  curl -d 'client_id=admin-cli' -d 'username=consumer-user' -d 'password=consumer-user' -d 'grant_type=password'     'https://kc-one.tim.apps.fiware.dev/realms/fiware-server/protocol/openid-connect/token' | \
    python -m json.tool


    curl -H "Authorization: Bearer 12345" https://kc-one.tim.apps.fiware.dev/realms/fiware-server/verifiable-credential/did:key:z6MkigCEnopwujz8Ten2dzq91nvMjqbKQYcifuZhqBsEkH7g?type=BatteryPassAuthCredential


    {
  "type" : [ "VerifiableCredential", "BatteryPassAuthCredential" ],
  "@context" : [ "https://www.w3.org/2018/credentials/v1", "https://w3id.org/security/suites/jws-2020/v1" ],
  "id" : "urn:uuid:3b492104-dbe7-4a67-aaf5-8ee0d048b613",
  "issuer" : "did:key:z6MkigCEnopwujz8Ten2dzq91nvMjqbKQYcifuZhqBsEkH7g",
  "issuanceDate" : "2023-03-24T09:57:40Z",
  "issued" : "2023-03-24T09:57:40Z",
  "validFrom" : "2023-03-24T09:57:40Z",
  "expirationDate" : "2023-03-26T21:57:40Z",
  "credentialSchema" : {
    "id" : "https://raw.githubusercontent.com/FIWARE-Ops/batterypass-demonstrator/main/docs/schema.json",
    "type" : "FullJsonSchemaValidator2021"
  },
  "credentialSubject" : {
    "id" : "8dcda148-2e4e-4feb-99a9-eb1e5645496b",
    "familyName" : null,
    "firstName" : null,
    "roles" : null,
    "email" : "consumer-user@fiware.org"
  },
  "proof" : {
    "type" : "JsonWebSignature2020",
    "creator" : "did:key:z6MkigCEnopwujz8Ten2dzq91nvMjqbKQYcifuZhqBsEkH7g",
    "created" : "2023-03-24T09:57:40Z",
    "verificationMethod" : "did:key:z6MkigCEnopwujz8Ten2dzq91nvMjqbKQYcifuZhqBsEkH7g#z6MkigCEnopwujz8Ten2dzq91nvMjqbKQYcifuZhqBsEkH7g",
    "jws" : "eyJiNjQiOmZhbHNlLCJjcml0IjpbImI2NCJdLCJhbGciOiJFZERTQSJ9..JUaJimmYv0IfHGFBJcSWX8LfLRrKXVm83t6JYFYw3pEPqm1e1X-a-p-YnYo-rg1KcjN8ErAziZ7YDQNYuCCUBQ"
  }
}t