# Payer Identity in Web-based Payment Handlers

Author: danyao@chromium.org <br/>
Status: draft <br/>
Last updated: 2020-02-17

This explainer attempts to enumerate the technical solutions that are currently
available to web applications that implement the [Payment Handler API][ph-api]
to establish payer identity during a payment flow.

__A note to readers:__
Since I'm learning about web storage and privacy as I go, any feedback is
highly appreciated. 🙂🙏🏻 Please email me or file an issue in this repo.
The end goal of this document is to either confirm that
Payment Handler API has sufficient features to support the type of identity flow
needed during web payments, or identify any missing features that we should add
to the API.

## Background

### Role of a Payment Handler

In the [web payments architecture][architecture-overview], a *payment app* is a
piece of software that processes payment requests sent via the *mediator* (i.e.
browser) and returns payment responses to the mediator.
A *web-based payment handler* is one way to implement a payment app. It
integrates with the browser via the [Payment Handler API][ph-api]. The rest of
this document uses "payment app" and "payment handler" interchangeably.

The role of a payment app is essentially that of a wallet, which has four core
responsibilities during a payment flow:

1. __Provide a user interface__ for the user to choose an instrument from within
the wallet (where there may be only one) to pay for the current transaction.
1. __Authenticate the user__ as the owner of the financial account represented
by the wallet.
1. __Obtain authorization__ from the user (who has been proven to be the account
owner) to proceed with the payment.
1. __Execute the payment__ or generate the necessary data for another entity
(usually the merchant) to execute the payment.

Typically, the financial account represented by a payment app is a service that
the user already has an established relationship with prior to the start of the
current transaction. This may be a bank that has issued the user a credit card
or a debit card, or a digital store-of-value wallet that the user has preloaded
with funds. Hence the main goal of the authentication step is to prove to the
financial service provider (i.e. the bank or the digital store-of-value wallet
provider) that the current user in the browser is the same person who owns the
account.

### Payment Handler Lifecycle

A web-based payment handler is a [Service Worker][service-worker] that
[registers][ph-set] the capability to handle payment requests with the browser.
Its lifecycle includes the following stages:

1. __Pre-installation__: the service worker is not yet installed in the browser.
1. __Discovered__: the service worker has been identified as the default
application in a [Payment Method Manifest][payment-manifest], possibly as part
of a browser's Just-in-Time installation flow, but is not yet installed in the
browser.
1. __Installed__: the service worker is [installed][sw-install], and has
registered with the browser as a payment handler, but has not been linked to any
payer identity needed to identify the financial account backing this payment
handler.
1. __Enrolled__: the payment handler installed in the current browser has been
linked to a financial account. There may be two slight variations to this state:
  a. __Weak Enrolled__: this payment handler has knowledge of a payer identity (
     e.g. an email address), but cannot prove that it has been authorized by the
     account owner to act on behalf of the account.
  b. __Strong Enrolled__: this payment handler can present proof (e.g.
     side-channel or cryptographic proof) that the account owner has approved
     the association of this payment handler on this device to the financial
     account.

Achieving the Weak Enrolled state is the minimum requirement for a payment
handler to able to play its role during a payment transaction.

### hasEnrolledInstrument()

The [Payment Request API][pr-api] provides a `hasEnrolledInstrument()` method
that the merchant can used to query if a payment handler is in the enrolled
state so it can complete the payment with low user friction. The intuition here
is that a merchant may prefer to only offer a particular payment option if the
associated the payment handler can guarantee a good user experience.

The current [Payment Handler API][ph-api] design only allows a payment handler
to respond to `hasEnrolledInstrument()` using an event handler registered in the
service worker and without opening any user interface. As a result, the only
persistent storage that the service worker can leverage to store the enrolled
user identity is [IndexedDB][indexed-db].

## Enrollment Flows

### On-the-fly enrollment

This flow can be implemented today using existing technology:

__First time, unrecognized user:__

1. Merchant initiates payment request and indicates they support
   `https://payment-handler.example`.
1. `hasEnrolledInstrument()` returns `false` because no payment handler is
   installed.
1. User interacts with merchant website, which triggers `request.show()`.
1. Browser discovers the default payment handler just-in-time and presents it
   as a payment option to the user. The payment handler is in "Discovered"
   state.
1. User clicks on "Pay" in browser UI to proceed with the payment handler from
   `https://payment-handler.example`. This moves the payment handler into the
   "Installed" state.
1. Payment handler receives `PaymentRequestEvent` and calls
   `event.openWindow('https://payment-handler.example/ui')` to open up a web
   signin flow.
1. Browser loads web signin flow URL in a payment handler window.
1. User completes sign in inside the payment handler window.
1. `https://payment-handler.example/ui` persists a user ID in IndexedDB. At this
   point, the payment handler is in "Weak Enrolled" state.
1. (Optional) `https://payment-handler.example/ui` uses [WebAuthn][webauthn] to
   register the current device to the user's account. At this point, the
   payment handler is in "Strong Enrolled" state.

Here is an example sketch of how the web signin flow can persist the user ID
using IndexedDB:

```
// https://payment-handler.example/ui

function onUserSignin(userId) {
  let open = window.indexedDB.open("phDB", 1);

  // Create object store if DB is empty.
  open.onupgradeneeded = (evt) => {
    let db = evt.target.result;
    if (!db.objectStoreNames.contains("userstate")) {
      let store = db.createObjectStore("userstate", {keyPath: "id"});
      console.log("Created object store ", store);
    }
  };

  // Persists user ID.
  open.onsuccess = (evt) => {
    let db = open.result;
    let txn = db.transaction("userstate", "readwrite");
    let store = txn.objectStore("userstate");
		let request = store.add({
      id: 1234,
      userId: "someuserid"
    });

    request.onsuccess = (success) {
      console.log("User state persisted.");
    }

    request.onerror = (error) {
      // Handle error
    }
  }
}
```

Note that although the web signin flow can persists the user ID using cookies,
this path is not recommended because cookies are not accessible by service
workers, so would not be usable for the service worker to respond to
`hasEnrolledInstrument()` for subsequent visits.

__Returning user on the same browser:__

1. Merchant initiates payment request and indicates they support
   `https://payment-handler.example`.
1. Merchant calls `request.hasEnrolledInstrument()`. Since a payment handler is
   installed, the browser triggers `canmakepayment` event (to be renamed to
   `hasenrolledinstrument` event) in the payment handler's service worker.
  1. The service worker queries IndexedDB and finds a previously saved user ID,
     so it responds `true` to `canmakepayment` event.
  1. 'hasEnrolledInstrument()` resolves to true.
  > Note that the WebAuthn credential cannot be retrieved from the service
  > worker context.
1. User interacts with merchant website, which triggers `request.show()`.
1. Browser presents the installed payment handler as a payment option to the
   user (or skips directly to the payment handler if it is the only payment
   handler available).
1. Payment handler receives `PaymentRequestEvent` and calls
   `event.openWindow('https://payment-handler.example/ui')` to open up a web
   signin flow.
1. `https://payment-handler.example/ui` requests a WebAuthn assertion for the
   user ID to verify that this is the same device the user has previously
   enrolled.
1. Browser prompts the user for an authorization gesture, which the user
   provides.
1. `https://payment-handler.example/ui` verifies that this is the same device
   the user has previously registered.

Here is an example sketch of how the service worker can look up the user ID from
IndexedDB:

```
// https://payment-handler.example Service Worker

self.addEventListener("canmakepayment", (evt) => {
  let open = window.indexedDB.open("phDB", 1);

  open.onerror = (error) => {
    // Error opening the database.
    evt.respondWith(false);
  }

  open.onsuccess = (evt) => {
    let db = open.result;
    if (!db.objectStoreNames.contains("userstate")) {
      evt.respondWith(false);
      return;
    }

    let txn = db.transaction("userstate");
    let store = txn.objectStore("userstate");

    // Read the first record in the object store.
    let request = store.get(1);
    request.onsuccess = (getEvent) => {
      if (CheckUserID(request.result)) {
        evt.respondWith(true);
      }
    };

    request.onerror = (error) => {
      evt.respondWith(false);
    };
  };
});
```

### Explicit Enrollment on Payment Handler Origin

Current technology also allows a payment handler to enroll a user prior to the
first checkout to avoid friction during checkout. This requires the user to
visit the payment handler website explicitly.

__Explicit enrollment in payment handler origin:__

1. User visits `https://payment-handler.example` and signs in to an existing
   account using whatever method they have been using.
1. `https://payment-handler.example` installs a service worker and registers it
   as a payment handler with the browser.
1. `https://payment-handler.example/ui` stores a user ID in the IndexedDB. This
   puts the payment handler into "Weak Enrolled" state.
1. `https://payment-handler.example` initiates the creation of a new WebAuthn
   credential for the current user ID.
1. Browser prompts the user for an authorization gesture, which the user
   provides.
1. `https://payment-handler.example/ui` receives a public key credential, which
   it stores alongside the user ID in the backend. This puts the payment handler
   into "Strong Enrolled" state.

__User visits merchant checkout:__

The explicit enrollment flow achieves the same payment handler state as the
on-the-fly enrollment flow from the previous section, namely:

* A user ID is stored in IndexedDB that is accessible by the payment handler's
  service worker.
* `https://payment-handler.example` has a public key credential for the user. If
  the browser can prove the posession of the associated private key, then the
  relying party, i.e. `https://payment-handler.example`, knows that this is the
  same device that the user enrolled previously.

Hence, the same steps from "Returning user on the same browser" can be reused to
authenticate the user's identity.

## Special Note about SRC

The [Feb 5, 2020 Card Payment Security Task Force call][src-flow] discussed a
SRC user journey with the following key requirements:

* Browser installs just-in-time  a default "Click to pay - add a card" payment
  handler.
* The user enters a card (e.g. a Mastercard) and SRC ID (e.g. email) inside the
  default payment handler and gets redirected to a Mastercard website with
  SRC ID, still within the payment handler window.
* The Mastercard website triggers installation of a Mastercard payment handler.
* The Mastercard payment handler is linked to the user's identity (SRC ID and
  card number).
* The default payment handler is linked to the user's SRC ID.

These requirements are not fundamentally different from the general enrollment
flows described in this document and can be met with existing technology:

1. When user enters an SRC ID inside the default payment handler for the first
   time:
   1. The default payment handler can save the SRC ID into IndexedDB for later
      retrieval.
1. When a Mastercard payment handler is installed just-in-time:
   1. It can store the SRC ID it receives from the default payment handler into
      IndexedDB for later retrieval.
   1. It can optionally use WebAuthn to register the user's device with their
      SRC account and realize the "Strong Enrolled" state of the payment handler.

## New APIs

This section discuss potential new APIs that may be useful in streamlining the
user identify flows described above.

### More Persistent Storage than IndexedDB

The user states persisted in IndexedDB are deleted when the user explicitly
clears site data from the browser. The user will have to repeat the enrollment
steps to re-establish their identity with the payment handlers. It is possible
that the typical user may wish to persist their payment handler states in this
case (similar to how native app states persists on a device).

A possible solution is to extend the [PaymentManager][payment-manager] API with
a new read-only UUID attribute that uniquely identifies a payment handler
installation to replace the role of IndexedDB in the previous examples. A
payment handler can associate this UUID with a user identity on its server. This
UUID will be reset if the user explicitly removes a payment handler from their
browser.

Such a UUID can clearly be used as a tracker. To mitigate its privacy threat,
it should only be assigned as part of an explicit payment handler installation
flow.


## References

1. [Web Payments Overview 1.0][architecture-overview]
1. [Service Workers 1][service-worker]
1. [Payment Handler API][ph-api]
1. [Payment Request API][pr-api]
1. [Payment Method Manifest][payment-manifest]
1. [Indexed Database 2.0][indexed-db]
1. [Web Authentication Level 1][webauthn]
1. [Feb 5, 2020 Card Payment Security Task Force call on SRC flow][src-flow]

[ph-api]: http://w3c.github.io/payment-handler
[architecture-overview]: https://www.w3.org/TR/webpayments-overview/#roles-in-the-ecosystem
[service-worker]: https://www.w3.org/TR/service-workers/
[ph-set]: https://w3c.github.io/payment-handler/#registration
[payment-manifest]: https://w3c.github.io/payment-method-manifest/#manifest-example
[sw-install]: https://www.w3.org/TR/service-workers/#dfn-lifecycle-events
[pr-api]: https://w3c.github.io/payment-request/
[indexed-db]: https://www.w3.org/TR/IndexedDB/
[3p-ph]: https://github.com/w3c/payment-handler/issues/351#issuecomment-566642121
[webauthn]: https://www.w3.org/TR/webauthn/
[src-flow]: https://www.w3.org/2020/02/05-wpwg-minutes
[payment-manager]: https://w3c.github.io/payment-handler/#paymentmanager-interface
