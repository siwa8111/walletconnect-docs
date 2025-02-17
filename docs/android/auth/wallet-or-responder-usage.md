# Wallet / Responder Usage

We recommend looking at example implementations of Responder at our [Kotlin GitHub repository](https://github.com/WalletConnect/WalletConnectKotlinV2/tree/develop/auth/responder)

### **Initialize Auth Client**

To initialize the Auth client, initialize first a `CoreClient` in the Android Application class. It will need the application class,
the server URL, connection type and the application AppMetaData. Next, pass CoreClient to AuthClient initialize function.

```kotlin
val projectId = "" // Get Project ID at https://cloud.walletconnect.com/
val relayUrl = "relay.walletconnect.com"
val serverUrl = "wss://$relayUrl?projectId=${projectId}"
val appMetaData = Core.Model.AppMetaData(name = "Kotlin.Responder",
    description = "Kotlin AuthSDK Responder Implementation",
    url = "kotlin.responder.walletconnect.com",
    icons = listOf("https://raw.githubusercontent.com/WalletConnect/walletconnect-assets/master/Icon/Gradient/Icon.png"),
    redirect = "kotlin-responder-wc:/request"
)

CoreClient.initialize(relayServerUrl = serverUrl, connectionType = ConnectionType.AUTOMATIC, application = this,metaData = appMetaData)

AuthClient.initialize(init = Auth.Params.Init(core = CoreClient)) { error -> Log.e(tag(this), error.throwable.stackTraceToString()) }
```
For more context on how to initialize CoreClient, go to [CoreClient docs](../../android/core/installation.md) section.

---
### **AuthClient.ResponderDelegate**

The AuthClient needs a `AuthClient.ResponderDelegate` passed to it for it to be able to expose asynchronously updates sent from the Dapp / Requester.

```kotlin
object ResponderDelegate : AuthClient.ResponderDelegate {
    init {
        AuthClient.setResponderDelegate(this)
    }

    override fun onAuthRequest(authRequest: Auth.Event.AuthRequest, verifyContext: Auth.Event.VerifyContext) {
        // Triggered when Dapp / Requester makes an authorization request. Wallet / Responder should display message to user and ask him to approve or reject authorization.
    }

    override fun onConnectionStateChange(connectionStateChange: Auth.Event.ConnectionStateChange) {
        // Triggered whenever the connection state is changed
    }

    override fun onError(error: Auth.Event.Error) {
        //Triggered whenever the error occurs with Relay Server
    }
}
```

`Auth.Event.VerifyContext` provides domain verification information about an AuthRequest. It consists of the origin of a Dapp from where the request has been sent, a validation Enum that says whether the origin is `VALID`, `INVALID` or `UNKNOWN`, and the verify url server. 

```kotlin
data class VerifyContext(
    val id: Long,
    val origin: String,
    val validation: Model.Validation,
    val verifyUrl: String
)

enum class Validation {
    VALID, INVALID, UNKNOWN
}
```

---

## **Methods**

### **Authorization Request Approval**

To approve authorization request, sign message using `CacaoSigner.sign` which requires private key to sign `Cacao` object that needs to be passed to `Auth.Params.Respond.Result` object and send to Dapp / Requester.
`issuer` parameter describes what did responder authorizes. Example `iss` for Ethereum Mainnet: `did:pkh:eip155:1:0xb9c5714089478a327f09197987f16f9e5d936e8a`. More about `did:pkh` method [here](https://github.com/w3c-ccg/did-pkh/blob/main/did-pkh-method-draft.md).

```kotlin
val request: Auth.Event.AuthRequest = // Request from onAuthRequest
val signature: Auth.Model.Cacao.Signature = CacaoSigner.sign(
    request.message, // Message to be signed
    PRIVATE_KEY, // Private key used to signing a message
    SignatureType.EIP191 // or EIP1271
)
val issuer = //Check following link for more reference: https://github.com/w3c-ccg/did-pkh/blob/main/did-pkh-method-draft.md

AuthClient.respond(Auth.Params.Respond.Result(request.id, signature, issuer)) { error ->
    Log.e("Responder respond approve", error.throwable.stackTraceToString())
}
```

### **Authorization Request Rejection**

To reject authorization request respond Dapp / Requester with `Auth.Params.Respond.Error`. Note: We recommend using defined below error message and error code.

```kotlin
val request: Auth.Event.AuthRequest = // Request from onAuthRequest
AuthClient.respond(
    Auth.Params.Respond.Error(request.id, 12001, "User Rejected Request") // Specifying Error codes will change in future
) { error ->
    Log.e("Responder respond reject", error.throwable.stackTraceToString())
}
```

### **Format message**

To receive formatted SIWE message, call formatMessage method with following parameters:

```kotlin
val payloadParams: Auth.Params.PayloadParams = //PayloadParams received in the onAuthRequest callback
val issuer = //MUST be the same as send with the respond methods and follows: https://github.com/w3c-ccg/did-pkh/blob/main/did-pkh-method-draft.md
val formatMessage = Auth.Params.FormatMessage(event.payloadParams, issuer)

AuthClient.formatMessage(formatMessage)
```

### **Get List of Pending Requests**

```kotlin
val pendingRequest: List<Auth.Model.PendingRequest> = AuthClient.getPendingRequest()

```

To get a list of pending request, call `AuthClient.getPendingRequest()` which will return a list of type `Auth.Model.PendingRequest`.

### **Sample App**
To check more in details go and visit our responder implementation app [here](https://github.com/WalletConnect/WalletConnectKotlinV2/tree/develop/auth/responder)