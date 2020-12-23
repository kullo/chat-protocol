Kullo Chat Protocol
===================

## Basic types

### Password

The user-defined password or passphrase. Never persisted.

### LoginKey

A `pkdf(Password)` used for [authentication of requests](#request-authentication) to the server.

* Client-side: stored locally on every device
* Server-side: stored in a hashed way in order to verify authentication

### PasswordVerificationKey

A `pkdf(Password)` used when the user is asked to manually re-type the password.

* Client-side: never persisted
* Server-side: stored in a hashed way in order to verify password input

### EncryptionPrivkeyEncryptingKey

A `pkdf(Password)` used as a symmetric encryption key for `EncryptionPrivkey`.

* Client-side: stored locally on every device; never sent
* Server-side: –

### EncryptionKeypair

An asymmetric key pair used for encrypting data for the owner. This key pair can be rotated by the owner.

#### EncryptionPrivkey

* Client-side: generated or downloaded, encrypted before upload
* Server-side: stored for synchronization (symmetrically encrypted by the owner using `EncryptionPrivkeyEncryptingKey`)

#### EncryptionPubkey

* Client-side: downloaded by non-owners to encrypt for owner
* Server-side: published for all users to encrypt for owner

### DeviceKeypair

An asymmetric key pair used for [request authentication](#request-authentication) and message signing.

### DeviceKeypairId

The ID of a `DeviceKeypair`.
Implemented as a fingerprint of the `DevicePrivkey`.

#### DevicePrivkey

* Client-side: generated and stored; never sent
* Server-side: –

#### DevicePubkey

* Client-side: downloaded from non-owners to verify signatures
* Server-side: published for all users to allow verifying message signatures; used to verify request authentication

## Request authentication

When sending requests to the server, the client authenticates itself by proving the knowledge of the password and the ownership of a valid `DeviceKeypair`. This pair of credentials becomes invalid as soon as the user changes the password or the `DeviceKeypair` is [blocked](#blocking-a-device). To be able to cache the authentication data without revealing the plain password when a device is lost, the password is persisted and transmitted as the `LoginKey`. Ownership of `DeviceKeypair` is proven by signing the `LoginKey` with `DeviceKeypair`. This is a stateless authentication mechanism.

The user sending an authenticated request is called the "current user".

## Registration

Registration is an unauthenticated request to the server, sending
1. plain text email address
2. `LoginKey`
3. `PasswordVerificationKey`
4. `DevicePubkey`
5. `EncryptionPubkey`
6. encrypted `EncryptionPrivkey`

The server then
* checks if the email address is allowed for registration,
* stores a hashed version of `LoginKey` and `PasswordVerificationKey`,
* stores `DevicePubkey`, `EncryptionPubkey`,
* stores the encrypted `EncryptionPrivkey`,
* sets the user state to pending,
* (optional) sends an info email to the registering user and the admin,
* sends a server-generated registration verification code to the client to display.

At this point, everyone on the internet could have done this registration request by just knowing a valid unused company email address.

Optional DoS countermeasures could be implemented, e.g. limiting the client IP range directly or via IP-to-location restrictions, restrict client software using client credentials, invitation codes et cetera. Let's assume there is a very low number of pending registrations.

To **verify** the user, the admin must manually activate the pending registration. Technically this could be done by just switching the user state to active. However, the server software should require to type in the registration verification code shown to the user in order to be sure to not activate the wrong person.

When verified, the user can access the users list and open new channels. In order to join existing channels, the user must be invited.

## Registering a new device

A verified user may choose to use an additional device. The new device will share the same password (stored as `LoginKey`) with all other devices and generate a dedicated `DeviceKeypair`. The new device registration is an unauthenticated request to the server, containing:

* `PasswordVerificationKey` to prove the user knows the password
* `DevicePubkey`

The server then checks `PasswordVerificationKey` against the stored hashed `PasswordVerificationKey` and if valid, registers the `DevicePubkey` in pending state. The new device registration is verified manually by comparing a fingerprint of `DevicePubkey` either by the user on an existing device or by the admin.

## Logout

The logout operation is a combination of
1. deleting all locally persisted keys/cache,
2. blocking the current device.

In case the request in 2. fails, the device can be blocked from another device or an admin interface.

## Blocking a device

When a device is lost, its `DevicePubkey` can be blocked. Blocking is setting a server-side `DeviceBlockTime` from `null` to the time the device was lost. This timestamp may be in the past but not in the future, such that a non-null block timestamp means the `DevicePubkey` is valid. Using a past block timestamp makes former valid signatures (i.e. messages sent from a stolen device) invalid.

Authenticating requests using the blocked device's `DeviceKeypair` fails, such that the user cannot do anything with the device when blocked.

A device may be unblocked by setting the `DeviceBlockTime` back to `null`. This assumes the user did not send messages from the device while it was blocked. Note: The user interface must explain to the user that unblocking is only secure when you know the device never left a trusted area (e.g found hidden in the own flat) and is not secure when the device left your control (e.g. stolen devices handed out by the police), because the device key might be extracted.

If the device is blocked because it may have been compromised, the owner's `EncryptionKeypair` is marked as compromised and a [rotation](#key-rotation) of all `ConversationKey`s the owner has access to is scheduled.

## Password change

An authenticated request to the server containing the new
* `LoginKey`,
* `PasswordVerificationKey`,
* encrypted `EncryptionPrivkey`

as well as the old `PasswordVerificationKey` proving the user knows the old password.

This make all future requests from other devices invalid. The server sends a password invalid response allowing the user to understand the reason for failing requests and login again.

## Password reset

TODO: do

## Conversation types

### ConversationKey

A raw symmetric key used to encrypt messages in conversation.

### ConversationKeyId

The ID of a conversation's symmetric encryption key.
Implemented as a fingerprint of the key.

### ConversationKeyBundle

A tuple of
1. a `ConversationKey`,
2. a `ConversationKeyId`.

### ConversationHistory

A map from `valid_from` timestamps to `ConversationKeyBundle`.

### ConversationPermission

A tuple containing
1. a conversation ID,
2. an owner,
3. a creator,
4. a `ConversationHistory` asymmetrically encrypted from creator for owner.

* Client-side: download all ConversationPermission with owner = current user. This is the list of available conversations displayed in the user interface.
* Server-side: stored for the owner's use

## Inviting a user to a channel

When no further restrictions are made, every user in a channel can invite every other active user to that channel. Every channel has a shared `ConversationHistory` containing a list of symmetric `ConversationKey`. This `ConversationHistory` is sent asymmetrically encrypted for the invited user to the server where it is stored as a `ConversationPermission`.

Next time the invited user requests a channel list, `ConversationHistory` is available for every channel, so all messages can be decrypted and new messages can be sent using the latest `ConversationKey` for encryption.

Receiving an invalid `ConversationPermission` may harm the invited user in different ways, e.g.

- the most recent keys may be removed from the history, causing the invited user to encrypt with an outdated, potentially compromised key;
- any key may be removed, causing undecryptable messages for the invited user;
- two regular users broadcast key rotations at nearly same time, causing a deletion of the new key broadcasted earlier.

TODO: ensure integrity of `ConversationPermission`

## Key rotation

When a device is lost, the `EncryptionKeypair` is considered compromised. While this may leak information of messages in the past, key rotation restores security for future communication.

Key rotation is also required when a user leaves a channel, excluding him from future communication or when a user leaves the team, excluding him from all future communication.

There are two key types that may need rotation.

### ConversationKey rotation

The ConversationKey can be rotated by any trusted conversation member. A new `ConversationKey` is generated and posted on top of `ConversationHistory` and sent to all conversation members as a `ConversationPermission`. This process can happen highly asynchronously because other conversation members can continue to use the outdated key as long as they want for sending and can download the updated `ConversationPermission` as soon as they receive a message encrypted with the new key.

ConversationKey rotation should be performed as soon as possible after a conversation member loses a device and should not depend on any actions by the affected user, because he may not have a second device or may be on holidays.

The new `ConversationKey` is not sent to conversation members with a `EncryptionPubkey` marked as compromised.

TODO: define strategy which conversation member is responsible for doing the ConversationKey rotation.

### EncryptionKeypair rotation

When the current user's `EncryptionKeypair` is rotated, this usually means that the symmetric `ConversationKey`s encrypted with `EncryptionPubkey` must be rotated as well. This ConversationKey rotation is done independently and may happen before or after EncryptionKeypair rotation.

When a user with a compromised `EncryptionKeypair` asks the server for his `EncryptionKeypair`, server will ask the user to generate a new `EncryptionKeypair` instead.

The user has to re-encrypt his `ConversationPermission`s. This is done by opening the permissions using the old private key and encrypting them using the new public key.

## Message types

### MessageId

An ID for each message unique across all conversations.

This is a server-assigned integer ID in ascending order which allows fast client-side message sorting.

### Context

The context in which a message was sent, containing the previous message's `MessageId` and the `ConversationKeyId`.

This implicitly fixes the conversation as `MessageId` and `ConversationKeyId` do not occur in multiple conversations.

### PlainMessage

A plain text message.

### SignedMessage

A tuple `(Context, PlainMessage)` signed using the `DeviceKeypair`.

## Encrypting a message

A `SignedMessage` is encrypted symmetrically using the latest `ConversationKey`.

## Sending a message

1. Ensure the latest `ConversationKey` is available,
2. make a `Context` and a `SignedMessage`,
3. encrypt and
4. send.

When server rejects the message because previous `MessageId` changed, repeat the process starting at 2.

## Editing a message

1. Client sends an authenticated request with the updated content (`SignedMessage`) to override an existing message identified by it's `MessageId`,
2. server checks the ownership of the message,
3. server pushes a message changed event to the clients to trigger an update.

## Deleting a message

Client edits the message into a *deleted* state causing the clients to remove it from the user interface.

## Reactions

Every message has a list of reactions of type `SignedMessage`. A reaction is a message with a parent ID (`MessageId`). Reactions are managed by the server (i.e. reaction count is available when delivering a message). The list of reactions can be downloaded on demand in a separate request, asynchronously from the main stream of messages.
