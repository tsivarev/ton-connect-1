# Bridge API

Bridge is a transport mechanism to deliver messages from the app to the wallet and vice versa.

* **Bridge is maintained by the wallet provider**. App developers do not have to choose or build a bridge. Each wallet’s bridge is listed in the [wallets-list](https://github.com/ton-connect/wallets-list) config.
* **Messages are end-to-end encrypted.** Bridge does not see the contents or long-term identifiers of the app or wallet.
* **Communication is symmetrical.** Bridge does not distinguish between apps and wallets: both are simply clients.
* Bridge keeps separate queues of messages for each recipient’s **Client ID**.

Bridge comes in two flavors:

- [HTTP Bridge](#http-bridge): for the external apps and services.
- [JS Bridge](#js-bridge): for dapps opened within the wallet; or when the wallet is a browser extension.

## HTTP Bridge

Client with ID **A** connects to the bridge to listen to incoming requests.

**Client ID is semi-private:** apps and wallets are not supposed to share their IDs with other entities to avoid having their messages removed unexpectedly.

**Each Client ID occupies only one connection.** If multiple wallet instances are connected with the same Client ID, the latest connection takes precedence. We can later add support for multiple instances through a proxy API or via a different bridge logic.

```tsx
request
	GET /events?client_id=<A>

	Accept: text/event-stream
```

Sending message from client A to client B. Bridge returns error if ttl is too high.

```tsx
request
	POST /message?client_id=<A>?to=<B>&ttl=300

	body: Message
```

Bridge buffers messages up to TTL (in secs), but removes them as soon as the recipient receives the message.

Bridge transfers message `{ from: A, message: Message }` to the client B.

## JS bridge

Used by the embedded apps via the injected binding `window.tonconnect`.

JS bridge runs on the same device as the wallet and the app, so communication is not encrypted.

The app works directly with plaintext requests and responses, without session keys and encryption.

```tsx
interface TonConnectBridge {
  deviceInfo(): DeviceInfo; // see Requests/Responses spec
  protocolVersion(): number; // max supported Ton Connect version (e.g. 2)
  connect(protocolVersion: number, message: InitialRequest, auto: boolean);
	send(message: AppRequest);
	listen((event: WalletResponse) => void);
}
```

Just like with the HTTP bridge, wallet side of the bridge does not receive the app requests except for **InitialRequest** until the session is confirmed by the user. Technically, the messages arrive from the webview into the bridge controller, but they are silently ignored.

SDK around the implements **autoconnect()** and **connect()** as silent and non-silent attempts at establishing the connection.

### connect(ver, req, auto)

Initiates connect request, this is analogous to QR/link when using HTTP bridge.

If the app was previously approved for the current account — connects silently with InitialReply.

If the app was not previously approved:

- `auto == true`: silently ignores this request.
- `auto == false`: shows confirmation UI. Wallet may bounce repeated requests to prevent DoS.

### **send(req)**

Sends the message to the bridge, including the InitialRequest (that goes into QR code when using HTTP bridge).

### listen(callback)

Registers a listener for events from the wallet. 

When the connection is being established, or when wallet switches to another account (previously approved), it sends **InitialReply** event with the info about the wallet.