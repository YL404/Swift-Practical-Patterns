# App Attest registration and protected requests

Use this reference when an iOS app's backend needs evidence about the app instance making a sensitive request. App Attest supplements the application's login, authorization, and HTTPS transport. The server makes the trust decision; a successful client API call alone does not authorize access.

Verified against Apple documentation on 2026-10-25. The client API requires iOS/iPadOS 14; this reference covers the main app, not extension or macOS behavior. Use a registered App ID, configure the App Attest capability/environment, and check `DCAppAttestService.shared.isSupported` before key operations. An unsupported client needs an explicit server policy; it must not silently become a verified client. [Client integration](https://developer.apple.com/documentation/devicecheck/establishing-your-app-s-integrity), [environment entitlement](https://developer.apple.com/documentation/bundleresources/entitlements/com.apple.developer.devicecheck.appattest-environment)

## Register a key once for each user on each device

After checking support, call `generateKey()` and persist the returned identifier immediately. The private key remains in the Secure Enclave; losing its identifier prevents later use. Keep registration state distinct from merely having a key identifier. Do not share one key among user accounts. Normal app updates retain keys; reinstallation, migration, and backup restoration require a new registration. [Key ownership and lifecycle](https://developer.apple.com/documentation/devicecheck/establishing-your-app-s-integrity)

1. Obtain a random, unique, one-time challenge from the backend. Apple specifies at least 16 bytes for the challenge. The server retains the expected value for verification.
2. Hash that challenge with SHA-256, then call `attestKey(_:clientDataHash:)` with the stored key identifier.
3. Send the key identifier and opaque attestation object to the server. Only after server verification succeeds should the app consider that key registered for subsequent assertions.

For `attestKey` returning `DCError.serverUnavailable`, retry later with the **same key and client-data hash**, rather than generating keys in a retry loop. Coordinate that retry with the server's pending challenge policy. Other attestation errors and server rejection require the documented key recovery path. An upload timeout after successful attestation is a different case: reconcile server registration before trying to attest the same key again; an already-attested key can produce `invalidKey`. [Attestation and retry contract](https://developer.apple.com/documentation/devicecheck/dcappattestservice/attestkey(_:clientdatahash:completionhandler:)), [invalidKey](https://developer.apple.com/documentation/devicecheck/dcerror-swift.struct/invalidkey)

## Bind assertions to the bytes the server will execute

After registration, obtain a fresh one-time challenge for a protected operation. Include it and the operation's meaningful parameters in `clientData`, hash those bytes, and call `generateAssertion`. Transmit those **same bytes** with the assertion and key identifier. Treat the signed body as the operation's authoritative input; do not verify one body and execute conflicting unsigned URL or header parameters. This byte-preserving request envelope is an implementation recommendation based on Apple's [assertion validation procedure](https://developer.apple.com/documentation/devicecheck/validating-apps-that-connect-to-your-server).

The following complete client helper definitions produce opaque evidence only. The caller supplies a server-issued challenge of at least 16 random bytes and, for an assertion, a key already accepted by the server. Network transport, key persistence, user authorization, retry/recovery, and the server verifier are deliberately outside these helpers.

```swift
import CryptoKit
import DeviceCheck
import Foundation

@available(iOS 14.0, *)
nonisolated func attestStoredKey(
    keyID: String,
    challenge: Data
) async throws -> Data {
    let service = DCAppAttestService.shared
    guard service.isSupported else {
        throw DCError(.featureUnsupported)
    }
    guard challenge.count >= 16 else {
        throw DCError(.invalidInput)
    }
    return try await service.attestKey(
        keyID,
        clientDataHash: Data(SHA256.hash(data: challenge))
    )
}

@available(iOS 14.0, *)
nonisolated func makeProtectedRequest(
    keyID: String,
    challenge: Data,
    resourceID: String
) async throws -> (clientData: Data, assertion: Data) {
    let service = DCAppAttestService.shared
    guard service.isSupported else {
        throw DCError(.featureUnsupported)
    }
    guard challenge.count >= 16 else {
        throw DCError(.invalidInput)
    }
    let clientData = try JSONEncoder().encode([
        "action": "downloadResource",
        "resourceID": resourceID,
        "challenge": challenge.base64EncodedString()
    ])
    let assertion = try await service.generateAssertion(
        keyID,
        clientDataHash: Data(SHA256.hash(data: clientData))
    )
    return (clientData, assertion)
}
```

If the transport wraps `clientData` in JSON, Base64-encode that `Data` and decode it on the server before hashing. Do not independently JSON-encode the decoded fields and assume dictionary order or escaping will reproduce the signed bytes. The server must also parse and authorize the signed action/resource for the authenticated user. Assertions are normally reserved for sensitive operations; they are not a replacement for the account session.

## Keep verification and replay state on the server

Follow Apple's full [server validation procedure](https://developer.apple.com/documentation/devicecheck/validating-apps-that-connect-to-your-server) and use the [validation guide](https://developer.apple.com/documentation/devicecheck/attestation-object-validation-guide) to cross-check a verifier. This reference supplies no CBOR, certificate, or signature implementation.

- For registration, validate the certificate chain to Apple's App Attest root, challenge-bound nonce, public-key hash/key identifier, App ID hash, initial zero counter, environment, and credential identifier. The App ID uses the actual **App ID prefix** plus bundle identifier; the prefix is usually the Team ID, not an unconditional synonym for it. Store only verified key/user/device/environment associations and the receipt required by the fraud-assessment flow.
- For assertions, validate the signature using that stored key, the App ID hash, the expected one-time challenge, and a counter strictly greater than the last accepted value (greater than zero for the first assertion). Store the accepted counter. A valid signature does not waive challenge, counter, or user-authorization checks.
- As an implementation requirement derived from one-time challenges and increasing counters, make challenge consumption and the counter compare/update atomic against concurrent requests. Define how the protected operation handles retries. A later counter arriving first can cause an earlier in-flight request to fail; if the product needs both operations, coordinate their entire request/response path per key. Async functions alone do not provide this coordination.

This is an integration outline, not a complete verifier checklist for every OS. Apple's current server procedure additionally discusses launch-validation category and bundle-version metadata. Their exact field encoding, required-presence policy across older clients, and new-OS fixtures are outside this reference. Development `aaguid` naming also differs between Apple's preparation and validation pages; resolve the target service's expected value before implementing that allowlist, rather than accepting multiple values as a workaround. These gaps must be resolved before deploying a verifier based on this outline.

## Separate environments and bound rollout

Keep development and production key/receipt records separate. TestFlight, App Store, and Enterprise distribution use production regardless of a development entitlement. Stagger attestation onboarding and handle throttling; an ordinary update must not cause every launch to generate another key. [Environment behavior](https://developer.apple.com/documentation/bundleresources/entitlements/com.apple.developer.devicecheck.appattest-environment), [rollout guidance](https://developer.apple.com/documentation/devicecheck/preparing-to-use-the-app-attest-service)

## Validation scope

The exact Swift helpers passed typechecking with Xcode 27.1 (27A9275), Apple Swift 6.4 (swiftlang-6.4.0.34.1), and the iPhoneOS 27.1 SDK targeting arm64 iOS 14. Checks used Swift 6, complete strict concurrency, warnings as errors, and both `nonisolated` and `MainActor` default isolation. The functions are explicitly `nonisolated` and hold no shared registration state; they do not implement per-key sequencing.

No signed app, Secure Enclave operation, Apple attestation exchange, backend verifier, fraud assessment, or device lifecycle was run. Challenge replay, altered signed bytes, concurrent/out-of-order assertions, lost registration responses, reinstall recovery, unsupported-device policy, production rollout, and OS-specific metadata still require integration tests. Typechecking does not establish attestation success or server security.

## Sources

- https://livsycode.com/best-practices/apple-app-attest-in-ios-development/
