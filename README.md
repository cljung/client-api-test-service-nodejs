# client-api-test-service-nodejs
Azure AD Verifiable Credentials node.js sample that uses the new private preview VC Client APIs.

## Two modes of operations
This sample can work in two ways:
- As a standalone WebApp with it's own web UI that let's you issue and verify DID Verifiable Credentials.
- As an API service that works in combination with Azure AD B2C in order to use VCs as a way to authenticate to B2C.

## VC Client API, what is that?

Initially, Microsoft provided a node.js SDK for building Verifiable Credentials applications. Going forward, it will be replaced by an API, since an API makes implementation easier and also is programming language agnostic. Instead of understanding the various functions in the SDK, in the programming language you use, you only have to understand how to format JSON structures for issuing or verifying VCs and call the VC Client API. 

![API Overview](media/api-overview.png)

## Issuance

### Issuance JSON structure

To call the VC Client API to start the issuance process, the DotNet API creates a JSON structure like below. 

```JSON
{
    "authority": "...set at runtime...",
    "includeQRCode": false,
    "registration": {
        "clientName": "...set at runtime..."
    },
    "callback": {
        "url": "...set at runtime...",
        "nounce": "...set at runtime...",
        "state": "...set at runtime...",
        "headers": {
            "my-api-key": "blabla"
        }
    },
    "issuance": {
        "type": "...set at runtime...",
        "manifest": "https://beta.did.msidentity.com/v1.0/3c32ed40-8a10-465b-8ba4-0b1e86882668/verifiableCredential/contracts/VerifiedCredentialExpert",
        "pin": {
            "value": "",
            "length": 0
        }
    }
}
```

**Note** - The only thing you need to update before running the issuer sample is the manifest link, because the sample starts with downloading that manifest and uses information in it, such as `type` and `authority`. The `callback` attributes are set per issuance invocation.

- **authority** - is the DID identifier for your registered Verifiable Credential i portal.azure.com.
- **includeQRCode** - If you want the VC Client API to return a `data:image/png;base64` string of the QR code to present in the browser. If you select `false`, you must create the QR code yourself (which is not difficult).
- **registration.clientName** - name of your app which will be shown in the Microsoft Authentictor
- **callback.url** - a callback endpoint in your DotNet API. The VC Client API will call this endpoint when the issuance is completed.
- **callback.state** - A state value you provide so you can correlate this request when you get callback confirmation
- **callback.headers** - Any HTTP Header values that you would like the VC Clint API to pass back in the callbacks. Here you could set your own API key, for instance
- **issuance.type** - the name of your credentialType. Usually matches the last part of the manifest url
- **issuance.manifest** - url of your manifest for your VC. This comes from your defined Verifiable Credential in portal.azure.com
- **issuance.pin** - If you want to require a pin code in the Microsoft Authenticator for this issuance request. This can be useful if it is a self issuing situation where there is no possibility of asking the user to prove their identity via a login. If you don't want to use the pin functionality, you should not have the pin section in the JSON structure. The appsettings.PinCode.json contains a settings for issuing with pin code.
- **issuance.claims** - optional, extra claims you want to include in the VC.

In the response message from the VC Client API, it will include it's own callback url, which means that once the Microsoft Authenticator has scanned the QR code, it will contact the VC Client API directly and not your node.js code. The node.js code will get confirmation via the callback.

### Issuance Callback

In your callback endpoint, you will get a callback with the below message when the QR code is scanned.

```JSON
{"code":"request_retrieved","requestId":"9463da82-e397-45b6-a7a2-2c4223b9fdd0", "state": "...what you passed as the state value..."}
```

## Verification

### Verification JSON structure

To call the VC Client API to start the verification process, the DotNet API creates a JSON structure like below. Since the WebApp asks the user to present a VC, the request is also called `presentation request`.

```JSON
{
    "authority": "...set at runtime...",
    "includeQRCode": false,
    "registration": {
      "clientName": "...set at runtime..."
    },
    "callback": {
      "url": "...set at runtime...",
      "nonce": "...set at runtime...",
      "state": "...set at runtime...",
      "headers": {
        "my-api-key": "blabla"
      }
    },
    "presentation": {    
      "includeReceipt": true,
      "requestedCredentials": [
        {
          "type": "...set at runtime...",
          "manifest": "https://beta.did.msidentity.com/v1.0/3c32ed40-8a10-465b-8ba4-0b1e86882668/verifiableCredential/contracts/VerifiedCredentialExpert",
          "purpose": "the purpose why the verifier asks for a VC",
          "trustedIssuers": [ "did-of-the-Issuer-trusted" ]
        }
      ]
    }
  }
```

Much of the data is the same in this JSON structure, but some differences needs explaining.

- **authority** vs **trustedIssuers** - The Verifier and the Issuer may be two different entities. For example, the Verifier might be a online service, like a car rental service, while the DID it is asking for is the issuing entity for drivers licenses. Note that `trustedIssuers` is a collection of DIDs, which means you can ask for multiple VCs from the user
- **presentation** - required for a Verification request. Note that `issuance` and `presentation` are mutually exclusive. You can't send both.
- **requestedCredentials** - please also note that the `requestedCredentials` is a collection too, which means you can ask to create a presentation request that contains multiple DIDs.
- **includeReceipt** - if set to true, the `presentation_verified` callback will contain the `receipt` element.

**Note** - Same thing applies to this JSON structure where the only thing you need to update is the `manifest`. However, if the issuer is someone ole than your own credential, you need to add your DID as the `authority`. The `trustedIssuers` will be filled in from the `manifest`. If you are verifying a VC you issued yourself, you can leave `authority` blank, because the sample will set it if it doesn't start with `did:ion:`.

### Verification Callback

In your callback endpoint, you will get a callback with the below message when the QR code is scanned.

When the QR code is scanned, you get a short callback like this.
```JSON
{"code":"request_retrieved","requestId":"c18d8035-3fc8-4c27-a5db-9801e6232569", "state": "...what you passed as the state value..."}
```

Once the VC is verified, you get a second, more complete, callback which contains all the details on what whas presented by the user.

```JSON
{
    "code":"presentation_verified",
    "requestId":"c18d8035-3fc8-4c27-a5db-9801e6232569",
    "state": "...what you passed as the state value...",
    "subject": "did:ion: ... of the VC holder...",
    "issuers": [
      "type": [
            "VerifiableCredential",
            "your credentialType"
      ],
      "claims": {
        "displayName":"Alice Contoso",
        "sub":"...",
        "tid":"...",
        "username":"alice@contoso.com",
        "lastName":"Contoso",
        "firstName":"alice"
      }
    ],
    "receipt":{
        "id_token": "...JWT Token of VC..."
        },
        "state":"...VC Client API state..."
    }
}
```
Some notable attributes in the message:
- **claims** - parsed claims from the VC
- **receipt.id_token** - the DID of the presentation

## Running the sample

### Standalone
To run the sample standalone, just clone the repository, compile & run it. It's callback endpoint must be publically reachable, and for that reason, use `ngrok` as a reverse proxy to read your app.

```Powershell
git clone https://github.com/cljung/client-api-test-service-nodejs.git
cd client-api-test-service-nodejs
cd issuer
npm install
node app.js ./issuance_request_config_v2.json
```

Then, open a separate command prompt and run the following command

```Powershell
ngrok http 8081
```

Grab, the url in the ngrok output (like `https://96a139d4199b.ngrok.io`) and Browse to it.

### Together with Azure AD B2C
To use this sample together with Azure AD B2C, you first needs to build it, which means follow the steps above. 

![API Overview](media/api-b2c-overview.png)

Then you need to deploy B2C Custom Policies that has configuration to add Verifiable Credentials as a Claims Provider and to integrate with this DotNet API. This, you will find in the github repo [https://github.com/cljung/b2c-vc-signin](https://github.com/cljung/b2c-vc-signin). That repo has a node.js issuer/verifier WebApp that uses the VC SDK, but you can skip the `vc` directory and only work with what is in the `b2c` directory. In the instructions on how to edit the B2C policies, it is mentioned that you need to update the `VCServiceUrl` and the `ServiceUrl` to point to your API. That means you need to update it with your `ngrok` url you got when you started the DotNet API in this sample. Otherwise, follow the instructions in [https://github.com/cljung/b2c-vc-signin/blob/main/b2c/README.md](https://github.com/cljung/b2c-vc-signin/blob/main/b2c/README.md) and deploy the B2C Custom Policies


### Docker build

To run it locally with Docker
```
docker build -t client-api-test-service-dotnet:v1.0 .
docker run --rm -it -p 5002:80 client-api-test-service-dotnet:v1.0
```

Then, open a separate command prompt and run the following command

```Powershell
ngrok http 5002
```

Grab, the url in the ngrok output (like `https://96a139d4199b.ngrok.io`) and Browse to it.

## appsettings.json

The configuration you have in the `appsettings.json` file determinds which CredentialType you will be using. If you want to use your own credentials, you need to update this file. In order to make it easy to shift between different configurations, the `AppSettings:ActiveCredentialType` points to which setting in appsetting.json that should be used. This way you can have multiple configurations, just change one line and then restart to test a new Verifiable Credential.
The setting `AppSetting:ActiveCredentialType` determinds which CredentialType that should be used when the program is running. 

At run-time, the program will use the settings you provide to build the JSON payload to send to the VC Client API for isuance and presentation.

The available settings in the repo are:

- **AppSettings.VerifiedCredentialExpert** - The sample VC from docs.microsoft.com
- **AppSettings.Cljungdemob2cMembership** - References my VC which issues from my test Azure AD B2C tenant 
- **AppSettings.FawltyTowers2Employee** - References my VC which issues from my test Azure AD Premium tenant
- **AppSettings.FawltyTowers2CampusPass** - References my VC which issues from my test Azure AD Premium tenant. It uses an id token hint and a pin code and no OIDC IDP
- **AppSettings.PinCode** - References a VC where you only need a pin code to issue yourself a VC


```JSON
{
  "Logging": {
    "LogLevel": {
      "Default": "Trace",
      "Microsoft": "Warning",
      "Microsoft.Hosting.Lifetime": "Information"
    }
  },
  "AllowedHosts": "*",
  "AppSettings": {
    "ActiveCredentialType": "FawltyTowers2CampusPass",
    "ApiEndpoint": "https://dev.did.msidentity.com/v1.0/abc/verifiablecredentials/request",
    "ApiKey": "MyApiKey",
    "UseAkaMs": false,
    "CookieKey": "state",
    "CookieExpiresInSeconds": 7200,
    "CacheExpiresInSeconds": 300,
    "client_name": "DotNet Client API Verifier"
  },
  "VerifiedCredentialExpert": {
    "PinCodeLength": 0,
    "didIssuer": "did:ion:EiAUeAySrc1qgPucLYI_ytfudT8bFxUETNolzz4PCdy1bw:eyJkZWx0YSI6eyJwYXRjaGVzIjpbeyJhY3Rpb24iOiJyZXBsYWNlIiwiZG9jdW1lbnQiOnsicHVibGljS2V5cyI6W3siaWQiOiJzaWdfMjRiYjMwNzQiLCJwdWJsaWNLZXlKd2siOnsiY3J2Ijoic2VjcDI1NmsxIiwia3R5IjoiRUMiLCJ4IjoiRDlqYUgwUTFPZW1XYVVfeGtmRzBJOVoyYnctOFdLUFF2TWt2LWtkdjNxUSIsInkiOiJPclVUSzBKSWN0UnFQTHRCQlQxSW5iMTdZS29sSFJvX1kyS0Zfb3YyMEV3In0sInB1cnBvc2VzIjpbImF1dGhlbnRpY2F0aW9uIiwiYXNzZXJ0aW9uTWV0aG9kIl0sInR5cGUiOiJFY2RzYVNlY3AyNTZrMVZlcmlmaWNhdGlvbktleTIwMTkifV0sInNlcnZpY2VzIjpbeyJpZCI6ImxpbmtlZGRvbWFpbnMiLCJzZXJ2aWNlRW5kcG9pbnQiOnsib3JpZ2lucyI6WyJodHRwczovL2RpZC53b29kZ3JvdmVkZW1vLmNvbS8iXX0sInR5cGUiOiJMaW5rZWREb21haW5zIn1dfX1dLCJ1cGRhdGVDb21taXRtZW50IjoiRWlBeWF1TVgzRWtBcUg2RVFUUEw4SmQ4alVvYjZXdlZrNUpSamdodEVYWHhDQSJ9LCJzdWZmaXhEYXRhIjp7ImRlbHRhSGFzaCI6IkVpQ1NvajVqSlNOUjBKU0tNZEJ1Y2RuMlh5U2ZaYndWVlNIWUNrREllTHV5NnciLCJyZWNvdmVyeUNvbW1pdG1lbnQiOiJFaUR4Ym1ELTQ5cEFwMDBPakd6VXdoNnY5ZjB5cnRiaU5TbXA3dldwbTREVHpBIn19",
    "didVerifier": "did:ion:EiAUeAySrc1qgPucLYI_ytfudT8bFxUETNolzz4PCdy1bw:eyJkZWx0YSI6eyJwYXRjaGVzIjpbeyJhY3Rpb24iOiJyZXBsYWNlIiwiZG9jdW1lbnQiOnsicHVibGljS2V5cyI6W3siaWQiOiJzaWdfMjRiYjMwNzQiLCJwdWJsaWNLZXlKd2siOnsiY3J2Ijoic2VjcDI1NmsxIiwia3R5IjoiRUMiLCJ4IjoiRDlqYUgwUTFPZW1XYVVfeGtmRzBJOVoyYnctOFdLUFF2TWt2LWtkdjNxUSIsInkiOiJPclVUSzBKSWN0UnFQTHRCQlQxSW5iMTdZS29sSFJvX1kyS0Zfb3YyMEV3In0sInB1cnBvc2VzIjpbImF1dGhlbnRpY2F0aW9uIiwiYXNzZXJ0aW9uTWV0aG9kIl0sInR5cGUiOiJFY2RzYVNlY3AyNTZrMVZlcmlmaWNhdGlvbktleTIwMTkifV0sInNlcnZpY2VzIjpbeyJpZCI6ImxpbmtlZGRvbWFpbnMiLCJzZXJ2aWNlRW5kcG9pbnQiOnsib3JpZ2lucyI6WyJodHRwczovL2RpZC53b29kZ3JvdmVkZW1vLmNvbS8iXX0sInR5cGUiOiJMaW5rZWREb21haW5zIn1dfX1dLCJ1cGRhdGVDb21taXRtZW50IjoiRWlBeWF1TVgzRWtBcUg2RVFUUEw4SmQ4alVvYjZXdlZrNUpSamdodEVYWHhDQSJ9LCJzdWZmaXhEYXRhIjp7ImRlbHRhSGFzaCI6IkVpQ1NvajVqSlNOUjBKU0tNZEJ1Y2RuMlh5U2ZaYndWVlNIWUNrREllTHV5NnciLCJyZWNvdmVyeUNvbW1pdG1lbnQiOiJFaUR4Ym1ELTQ5cEFwMDBPakd6VXdoNnY5ZjB5cnRiaU5TbXA3dldwbTREVHpBIn19",
    "manifest": "https://beta.did.msidentity.com/v1.0/3c32ed40-8a10-465b-8ba4-0b1e86882668/verifiableCredential/contracts/VerifiedCredentialExpert",
    "credentialType": "VerifiedCredentialExpert",
    "client_logo_uri": "https://didcustomerplayground.blob.core.windows.net/public/VerifiedCredentialExpert_icon.png",
    "client_tos_uri": "https://www.microsoft.com/servicesagreement",
    "client_purpose": "To check if you know how to use verifiable credentials."
  },
  "SomeOtherCredentialType": {
  }
}
``` 

### LogLevel Trace

The sample is quite verbose and you should see trace information about every GET/POST that happens. This is for training purposes so you can follow what happens. 