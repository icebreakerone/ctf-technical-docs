# Energy Data Providers (EDPs)

## First steps

Refer to the [Common Technical Guide](common-technical-guide.md) for:

* an architectural overview of Perseus
* links to demo applications
* instructions for registering for the Core Trust Framework sandbox member portal 
* how to configure your application and issue it digital certificates

## Permission flow

Sequence diagrams for the user permission flows and technical details of the FAPI2 OAuth2 flow are available at 

[https://github.com/icebreakerone/perseus-sequence-diagrams](https://github.com/icebreakerone/perseus-sequence-diagrams)

## Configure API endpoint for Perseus and FAPI

Your API endpoint must follow the FAPI specification and follow the specifications for  certificates.

* Configure web service cryptography to meet the [Baseline TLS Configuration](https://specification.docs.ib1.org/baseline-tls-configuration/1.0/)
* Install a non-wildcard server certificate issued by a well-trusted public CA.
* Client Certificates
    * Verify the connecting client using the Sandbox Core TF root client CA certificate. This will be provided by IB1.
    * Check the role for the client application encoded in its certificate is `https://registry.core.sandbox.trust.ib1.org/scheme/perseus/role/carbon-accounting-provider`. Return 403 if it is not.
        * This URL should be configurable depending on the deployment environment because it will be different in production
    	* See [Member Identity Digital Certificates - Verifying a Client Certificate](https://specification.trust.ib1.org/member-identity-digital-certificates/1.0/#verifying-a-client-certificate)
    * Python helper library: https://pypi.org/project/ib1-directory/ 

## OAuth issuer

Implement the issuer side of the [OAuth with Member Identity Certificates](https://specification.docs.ib1.org/oauth-with-member-identity-certificates/1.0/.) flow.

The key points of the OAuth flow are the use of RFC9126 Pushed Authorization Request (PAR), the mTLS client certificate to identify the client rather than a `client_id`, and use of `response_type=code` & swapping the code for refresh and access tokens after authentication.

Perseus adds the following requirements to the underlying [OAuth with Member Identity Certificates](https://specification.docs.ib1.org/oauth-with-member-identity-certificates/1.0/.) specification:

* Verify the client certificate as described in [Configure API endpoint for Perseus and FAPI](#configure-api-endpoint-for-perseus-and-fapi)
* When the client submits a PAR, they are asserting that they have the necessary permission from the account holder to use the data within Perseus and send a derived report to a Financial Service Provider. Log this request in sufficient detail to satisfy an audit.
* The “authentication” step in your OAuth flow should request the details needed to identify the meter, as a one shot process. You should not ask the user to create an account or ask for contact details unless strictly required to identify the meter or meet regulatory requirements – errors discovered after the information has been submitted will be handled manually.

## Generate Provenance records

[Provenance](https://specification.trust.ib1.org/provenance-records/1.0/) Records are a series of “steps” which describe where data came from and how it has been processed. Each participant signs the record with their signing certificate, issued by the Directory. Steps may include assurance metadata, in this case, whether consumption data is from a meter or it has been estimated.

Because of the signatures, Provenance records are a complex format with JWS signatures and base64 encoded JSON data. A Python reference library is available to generate and update Provenance records, and example code to update the Provenance Records for Perseus is available at [https://github.com/icebreakerone/provenance/tree/perseus-scripts/perseus-scripts](https://github.com/icebreakerone/provenance/tree/perseus-scripts/perseus-scripts). This can be used directly in Python applications, or in an external process.

Your implementation must create a new Provenance record when serving consumption data to a CAP:

* Create a “Permission” step to record the permission from the user

```json
{
    "type": "permission",
    "scheme": "https://registry.core.sandbox.trust.ib1.org/scheme/perseus",
    "timestamp":  "2025-09-01-T12:34:56Z",
    "account": id_of_permission_step,
    "allows": {
        "licenses": [
            "https://smartenergycodecompany.co.uk/documents/sec/consolidated-sec/",
       "https://registry.core.sandbox.trust.ib1.org/scheme/perseus/licence/energy-consumption-data/2024-12-05"
        ]
    },
    "expires": "2026-09-01T12:34:56Z"
}
```

* Create an “Origin” step for the data retrieved from SmartDCC

```json
{           
    "type": "origin",
    "scheme": "https://registry.core.sandbox.trust.ib1.org/scheme/perseus",
    "sourceType": "https://registry.core.sandbox.trust.ib1.org/scheme/perseus/source-type/Meter",
    "origin": "https://www.smartdcc.co.uk/",
    "originLicence": "https://smartenergycodecompany.co.uk/documents/sec/consolidated-sec/",
    "external": true,
    "permissions": id_of_permission_step,
    "perseus:scheme": {
         "meteringPeriod": {
            "from": "2024-09–01T00:00:00",
            "to": "2025-09–01T00:00:00"
        }
    },
    "perseus:assurance": {
        "missingData": "https://registry.core.sandbox.trust.ib1.org/scheme/perseus/assurance/missing-data/Complete",
         "originMethod": "https://registry.core.sandbox.trust.ib1.org/scheme/perseus/assurance/origin-method/SmartDCCOtherUser"
  },
}
```

* Create a “Transfer” step with the ID of the client

```json
{
    "type": "transfer",
    "scheme": "https://registry.core.sandbox.trust.ib1.org/scheme/perseus",
    "of": id_of_origin_step,
    "to": url_of_client_in_directory,
    "standard": "https://registry.core.sandbox.trust.ib1.org/scheme/perseus/standard/energy-consumption-data/2024-12-05",
    "license": "https://registry.core.sandbox.trust.ib1.org/scheme/perseus/licence/energy-consumption-data/2024-12-05",
    "service": "https://dev.example.com/perseus-api/1.0",
    "path": "/readings",
    "parameters": {
        "measure": "import",
        "from": "2024-09–01T00:00:00",
        "to": "2025-09–01T00:00:00"
    },
    "permissions": id_of_permission_from_cap,
    "transaction": fapi_transaction_id
}
```

* Sign the complete record.

The steps added are simple data structures. You will be able to hard code most of the values, and the variable data is easy to obtain identifiers and timestamps.

The URLs will vary across environments. License and standard URLs need to be configurable in their entirety. Other identifiers vary by the hostname only, so you only need to include the Registry hostname in your configuration.

## Implement consumption data API

Implement the [Energy Consumption Data API](https://registry.core.sandbox.trust.ib1.org/scheme/perseus/standard/energy-consumption-data/2026-03-12) ([Open API specification](https://registry.core.sandbox.trust.ib1.org/scheme/perseus/api/consumption-data@2026-03-12.json#ed6faea19fe0b57ff60fb6eaf3d4ba626069aef1af59eeaf2bcd0b344c23a7af)), authenticated using the OAuth token. 


An OpenAPI viewer of the consumption API is available on the Registry
[https://registry.core.sandbox.trust.ib1.org/openapi-viewer/api.html?a=/scheme/perseus/api/consumption-data@2026-03-12.json](https://registry.core.sandbox.trust.ib1.org/openapi-viewer/api.html?a=/scheme/perseus/api/consumption-data@2026-03-12.json)

The response must include a signed Provenance record, generated as above.

**For the Perseus sandbox, this API must only return synthetic data.**

## Withdrawal of Permission

Implement [Withdrawal of Permission](https://specification.trust.ib1.org/withdrawal-of-permission/1.0/):

1. Implement an RFC7009 token revocation endpoint for your OAuth Issuer. When the CAP uses this endpoint, your implementation must immediately invalidate the token and delete any stored data.
1. If your implementation includes user registration/account creation (for example, if you provide alternative metering), you must implement a UI to list and withdraw Permissions for logged in users. When a user withdraws permission, invalidate the OAuth tokens, and send a withdrawal message to the CAP’s [Message Delivery](https://specification.trust.ib1.org/message-delivery-to-applications/1.0/) endpoint.

# Beyond “Perseus-Ready” - considerations for production

## Implement transaction logging
Log all interactions with peers to meet auditing requirements.

## Implement retry logic for consumption data

In cases where there is a delay in the consumption data being available (for example due to data retrieval via SmartDCC), the consumption API must return a [202 response](https://registry.core.pilot.trust.ib1.org/openapi-viewer/api.html?a=/scheme/perseus/api/consumption-data@2024-12-05.json#/Consumption/getConsumptionData). CAPs will implement retry logic to repeat the request.

While you can implement and test logic to retry fetching consumption data in the sandbox, you must implement it for the production environment.

## Ensure your production app uses production Registry URLs

The production registry will be published at `https://registry.core.trust.ib1.org/scheme/perseus`

The URLs to check are:

* Role URLs used to validate CAP client connections
* licence, standard and scheme identifiers in provenance records

## Move to Production environment

To deploy your application in production:

* Wait for your Member record to be created in the Production Directory, after signing the Perseus Agreement.
* Create your Application in the Directory.
* Create Production Client and Signing Certificates.
* Configure deployment with Production Identifiers & Certificates
