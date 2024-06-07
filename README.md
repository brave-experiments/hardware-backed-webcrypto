# WebCrypto extension for hardware backed key management

## Authors:

- Kyle Den Hartog (Brave)

## Participate
- [Repo Issues](https://github.com/brave-experiments/hardware-backed-webcrypto/issues)

## Introduction

As cryptographic key management is becoming more useful on the web it has meant that many wallets, extensions, and browser features need to re-implement key management for various features in the web platform. Furthermore, many sites wish to manage keys as well but often times cannot rely on software backed implementations only. This has meant that there's a lot of re-implementation of basic cryptographic operations in many various places that all encounter their own various tradeoffs. The purpose of this WebCrypto extension is to act as a low level key management API so that it would be possible to have hardware backed keys as a low level primitive. From there various features on the web such as wallets and browser APIs like WebAuthn Level 3 could be built upon this generalized key management layer.

## Goals [or Motivating Use Cases, or Scenarios]

The overarching meta-goal of this proposal is to abstract key management away from application logic and encapsulate it within the WebCrypto API. Some specific use cases where this would be useful include:

- Cryptocurrency Wallet Extensions
- Digital Credential Wallet Extensions
- Browser Managed digital credential wallets
- Software defined passkey authenticators managed by a Browser
- Device Bound Session Credentials

## Non-goals

- Key synchronization

While key synchronization is a key feature to further enhance the use cases and features described above, it's not immediately necessary for the functionality of this WebCrypto interface extension. It's highly likely though that this moves in parallel to this given the growing importance for key synchronization in order to handle recovery flows more easily.

One example of a parallel piece of work for key synchronization that may be integrated with this work in the future is happening at the FIDO Alliance in the [Credential Provider Special Interest Group](https://www.youtube.com/watch?v=Mje6J2IMRTY). 

## Key Management API Extension

[For each related element of the proposed solution - be it an additional JS method, a new object, a new element, a new concept etc., create a section which briefly describes it.]

### Create a hardware backed key via WebCrypto API
```js
let keyPair = await window.crypto.subtle.generateKey(
  {
    name: "ECDSA",
    namedCurve: "P-256",
  },
  extractable: false,
  bindings: {
    hardwareBound: true,
    originBindings: ["example.com"],
    identifier: "123ABC",
    updatable: true
  },
  ["sign", "verify"],
);
```

In this scenario, we're suggesting a modification to the `extractable` property such that the old method where a boolean was provided would imply the same usage as before this extension. This allows for the API to be an opt in functionality for browsers. The additional `bindings` property allows for initial configurations for the key to be set and will become additional properties of the `CryptoKey` object. Here's some brief explanations of the different additional properties:

- `hardwareBound`: This one determines if the CryptoKey will be hardware bound. In the event that the `hardwareBound` is set to true, but extractable is set to false an error would be returned. The reason this is an optional configuration parameter rather than assuming whenever a `bindings` is include the CryptoKey will be hardware bound is because there may be a need to reuse the origin bindings for software bound keys. If this property is not set, it will default to `false`. Once set, it can no longer be updated.

- `originBindings`: This is a list of strings which determines which origins have access to this cryptographic key for usage and management. Once an origin is set here, it can fully control the key in the same way as any other origin listed. If this property is not set, it will default to the origin of the caller.

- `identifier`: The concept of an identifier here is to allow for the CryptoKey object to be referred to beyond the lifetime of the `CryptoKey` object returned by `generateKey`. This is especially useful when the key is being stored in an enclave or TPM where the key is not accessible to the application layer such that the key object can be handled by the application layer. One common pattern that I think should be expected is for an global identifier like a UUID or potentially a DID URL to be set here such that the key and cryptographic operations are performed by the browser, but the management, rotation, and usage are managed at a higher application layer such as a web wallet.

- `updatable`: This allows the original creator to limit whether or not a key can be updated beyond the original values set.

### Update Key (grant a new origin access to to the key to use it)
```js
await window.crypto.subtle.updateKey("123ABC", {
    originBindings: ["example.com", "acmecorp.com"],
    identifier: "3eac496b-1df8-4a4c-ac17-daf053d162d9" // changes from a local identifer specific to the origin to a global identifier
});
```

The design of this API is meant to build a permissions model around the origin such that the first origin must explicitly add the second origin. A user prompt is likely necessary in order to prevent cross site tracking via the identifier or keypair

### Use key to sign and verify a message
```js
const ecdsaParams = { name: "ECDSA", hash: "SHA-256" };
const keyIdentifier = "3eac496b-1df8-4a4c-ac17-daf053d162d9";
const message = "Hello World!";
const signature = await window.crypto.subtle.sign(ecdsaParams, keyIdentifier, message);
const verified = await window.crypto.subtle.verify(ecdsaParams, keyIdentifier, signature, message);

if(verified) {
  console.log(message);
}
```

In this API, any origin (such as `example.com` or `acmecorp.com`) with permission to access and use a key that's hardware backed can also opt to delete the key as well. If the key is shared across origins vai having permission granted, the `sign` call could be run by `example.com` and `verify` could be ran by `acmecorp.com` which could be useful for upgrading a bearer token to a Device Bound Session Credential all verified locally in a single page application.

### Delete key by identifier
```js
await window.crypto.subtle.deleteKey("3eac496b-1df8-4a4c-ac17-daf053d162d9");
```

In this API, any origin with permission to access and use a key that's hardware backed can also opt to delete the key as well.

### Managing keys used in higher level APIs
Further, key management capabilities such as managing a key used for a browser API might look like this:

```js

// Create a Passkey specifically for authentication purposes
const passkey = await navigator.credentials.create({
  publicKey: {
    challenge: new Uint8Array([1, 2, 3, 4]), // Example value
    rp: {
      name: "SimpleWebAuthn Example",
      id: "example.com"
    },
    user: {
      id: "3eac496b-1df8-4a4c-ac17-daf053d162d9",  // Example value
      name: "user@dev.com",
      displayName: "Passkey123",
    },
    pubKeyCredParams: [
      { alg: -7, type: "public-key" },   // ES256
    ],
    authenticatorSelection: {
      userVerification: "discouraged",
    },
  },
});

// Reuse the key to sign data outside an authentication flow
const ecdsaParams = const ecdsaParams = { name: "ECDSA", hash: "SHA-256" };
const message = "Hello World!";
const signature = await window.crypto.subtle.sign(ecdsaParams, passkey.rp.id, message);

// limit access of a key created via a high level API to limit side effects. This would be done via `updatable` being set to `false`
const expectedError = await window.crypto.subtle.updateKey("3eac496b-1df8-4a4c-ac17-daf053d162d9", {
    originBindings: ["example.com", "acmecorp.com"],
});

console.log(expectedError) // returns an error like "cannot update this key".

// delete the passkey
await window.crypto.subtle.deleteKey("3eac496b-1df8-4a4c-ac17-daf053d162d9");

```

In this case, we've created a browser backed passkey via the passkey registration API (the user would select browser profile in the consent flow) and then can re-use the key for additional flows. In this case, because the key is not updatable it can only be used within the origin that called the registration. This is done to limit side effects of the key being reused across origins and to limit the effects of key re-use across different signature flows.

## Detailed design discussion

### Permissions model currently grants unilateral authority across origins

In order to reduce the complexity of the permissions model at this layer, this design opted to grant all permissions based on the origin. It's theoretically possible for this API to have a more complex API in order to handle more fine-grained permissions. For example, the origin that generates the key could set a more complex model where instead of the `originBinding` property containing just an array of origins, it would instead be an array of objects that define a more fine grained access control model.

### Cross origin tracking needs mitigation

While a user permission would be the easy answer for preventing cross origin tracking based on a public key or key identifier, this doesn't seem like an ideal solution and further considerations is needed here. Handling this in a cross origin context means that it keeps this functionality very low level and likely requires user prompting and user consent. However, it's not clear that the permissions context to this will be understood since it won't always be clear what the key is used for.

The reason, this is necessary though is because in certain use cases such as the cryptocurrency wallet example, locking a key to a particular origin (in this case an extension origin) would limit the ability for the user to move between wallets without incurring fees to transfer assets between one account to another. In most, cases the answer would likely be to re-issue a credential or token such that a key remains isolated to an origin. This may also be possible in the case of cryptocurrency wallets, but would require changes to blockchain account logic.

### Should Extension be allowed to call this API on an origin?

If the API is meant to be origin isolated then we may need to limit extensions ability to call this. Otherwise malicious content scripts can be used to execute and bypass the permissions model here. Traditionally, this has not fit within the browser security model because installed extensions are considered trusted. However, this may be a time to consider adjusting this security model.

## Considered alternatives

One consideration that was [raised](https://github.com/w3c/webcrypto/issues/263#issuecomment-1743145628) by [@RByers](https://github.com/RByers) was to focus this more within higher level API designs. This would enable further hardware backed keys within browser APIs themselves. However, it likely wouldn't address many of the application level use cases which likely won't ever be addressed as web platform specific APIs. Hence, having an outlet via a low level extension in Web Crypto that can be built upon seemed more reasonable.

## Stakeholder Feedback / Opposition

At this time, this explainer has not been shared around enough to garner feedback from various stakeholders such as implementers or other browser vendors. Further consideration is needed and this section will be updated accordingly when it's more clear how other participants think about this.

## References & acknowledgements

Here's a non-exhaustive list of people who should be ackowledged for their discussion so far:

- WebCrypto API WG who first considered this and ultimately opted to not include this in the first version, but made note of their discussions in the spec.
- Anonymous Author for their writeup in [this issue](https://github.com/w3c/webcrypto/issues/263)
- Further participant feedback during the discussion on the issue linked above
