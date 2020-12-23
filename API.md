# Kullo Chat API


## REST

The REST API is only used for opening a WebSocket connection, for potentially large transfers (conversation lists, message lists) and for stuff that must be done without authentication.

All request bodies are sent as JSON (`Content-Type: application/json`).

**Authentication** is done via the `Authorization` HTTP header with custom type `KULLO_V1` and the parameters
 * `loginKey` (base64 encoded)
 * `signature`: signature of LoginKey (see CryptoAlgorithms for details)

Example:

```
Authorization: KULLO_V1 loginKey="AA6BRFjQXG39XhzaEJHZytIdOPOl2tt4nzgvEojP5Kk=", signature="f1fff53f4c66d7c5f6983fafb76db31b,NHD+Kx5Keu2iZYj7p4H3PaV9fNc0FxXjZaHdpw0Qf5xUtV5Ue3OCihckqN9d2b61isWi10AMxoJTktg14e2hAg=="
```

### Users

#### Register account

Does not require an `Authorization` header.

```
POST /users
```
```json
{
    "name": "John Doe",
    "email": "john.doe@example.com",
    "loginKey": "(base64-encoded data)",
    "passwordVerificationKey": "(base64-encoded data)",
    "encryptionPubkey": "(base64-encoded data)",
    "encryptionPrivkey": "(encrypted, base64-encoded data)"
}
```

##### Response

Returns `200 OK` on success:

```json
{
    "verificationCode": "music pear battery t-shirt",
    "user": {
        "id": 42,
        "state": "pending",
        "name": "John Doe",
        "picture": "http://example.com/image/7g97g.jpg",
        "encryptionPubkey": "(base64-encoded data)"
    }
}
```

#### Update user

```
PATCH /users/:user_id
```
```json
{
    "user": {
        "state": "active",
        "email": "john.doe@example.com",
        "name": "John Doe",
        "picture": "http://example.com/image/7g97g.jpg"
    },
    "permissions": [
        {
            "conversationId": "3eca5a5226a54134890bd6a648b54c04",
            "conversationKeyId": "961e57c49ac08a897349d862ccc3f2f2",
            "conversationKey": "(encrypted for user 42, base64 encoded)",
            "ownerId": 42,
            "creatorId": 1,
            "validFrom": "2018-01-01T11:11:11Z",
            "signature": "(signing device ID),(base64 encoded signature)"
        }
    ]
}
```

`permissions` and all fields in `user` are optional. The permissions' `ownerId` must be the user's ID.

#### Change EncryptionKey

```
PATCH /users/:user_id
```
```json
{
    "user": {
        "encryptionPubkey": "(base64-encoded data)",
        "encryptionPrivkey": "(encrypted, base64-encoded data)"
    },
    "permissions": [
        {
            "conversationId": "3eca5a5226a54134890bd6a648b54c04",
            "conversationKeyId": "961e57c49ac08a897349d862ccc3f2f2",
            "conversationKey": "(encrypted for user 42 using the new encryption key, base64 encoded)",
            "ownerId": 42,
            "creatorId": 1,
            "validFrom": "2018-01-01T11:11:11Z",
            "signature": "(signing device ID),(base64 encoded signature)"
        }
    ]
}
```

The `user` object contains the new `encryptionPubkey` and the new `encryptionPrivkey` encrypted using the current `EncryptionPrivkeyEncryptingKey`.
`permissions` is the full list of re-encrypted permissions, replacing all existing permissions with the same owner.

#### Change user password

```
POST /users/:user_id/change_password
```
```json
{
    "oldPasswordVerificationKey": "(base64-encoded data)",
    "user": {
        "loginKey": "(base64-encoded data)",
        "passwordVerificationKey": "(base64-encoded data)",
        "encryptionPrivkey": "(base64-encoded data)"
    }
}
```

The `user` object contains the changed `loginKey` and `passwordVerificationKey` and the newly encrypted `encryptionPrivkey`.

#### Get all users

```
GET /users?state=xyz
```

`state` (optional) is currently one of `pending` or `active`.

##### Response

Returns `200 OK` on success:

```json
{
    "objects": [
        {
            "id": 22,
            "state": "active",
            "name": "John Doe",
            "picture": "http://example.com/image/7g97g.jpg",
            "encryptionPubkey": "(base64-encoded data)"
        },
        {
            "id": 23,
            "state": "active",
            "name": "Jane Doe",
            "picture": "http://example.com/image/7g97g.jpg",
            "encryptionPubkey": "(base64-encoded data)"
        }
    ],
    "meta": {}
}
```

#### Get me

This endpoint can be used during the login to retrieve the user ID
required to sign a new device.

Does not require an `Authorization` header. Authentication is based on `email`/`passwordVerificationKey`.

```
POST /users/get_me
```
```json
{
    "email": "john.doe@example.com",
    "passwordVerificationKey": "(base64-encoded data)"
}
```

##### Response

Returns `200 OK` on success:

```json
{
    "user": {
        "id": 22,
        "state": "active",
        "name": "John Doe",
        "picture": "http://example.com/image/7g97g.jpg",
        "encryptionPubkey": "(base64-encoded data)"
    },
    "encryptionPrivkey": "(encrypted, base64-encoded data)"
}
```

Returns `403 Forbidden` if credentials do not match a user. This includes non-existing email addresses.

### Devices

#### Register a device

Does not require an `Authorization` header. Authentication is based on `email`/`passwordVerificationKey`.

```
POST /devices
```
```json
{
    "email": "john.doe@example.com",
    "passwordVerificationKey": "(base64-encoded data)",
    "device": {
        "id": "60a0a2b646e18247f97ded4e30a65fd0",
        "ownerId": 42,
        "idOwnerIdSignature": "60a0a2b646e18247f97ded4e30a65fd0,(base64-encoded data)",
        "pubkey": "(base64-encoded data)",
        "state": "pending",
        "blockTime": null
    }
}
```

##### Response

Returns `200 OK` on success:

```json
{
    "id": "60a0a2b646e18247f97ded4e30a65fd0",
    "ownerId": 42,
    "idOwnerIdSignature": "60a0a2b646e18247f97ded4e30a65fd0,(base64-encoded data)",
    "pubkey": "(base64-encoded data)",
    "state": "pending",
    "blockTime": null
}
```

Returns `409 Conflict` if a device with the given ID already exists.

#### Get all pending devices

```
GET /devices?state=pending
```

##### Response

Return `200 OK` on success, including the owners in `related`:

```json
{
    "objects": [
        {
            "id": "60a0a2b646e18247f97ded4e30a65fd0",
            "ownerId": 42,
            "idOwnerIdSignature": "60a0a2b646e18247f97ded4e30a65fd0,(base64-encoded data)",
            "pubkey": "(base64-encoded data)",
            "state": "pending",
            "blockTime": null
        }
    ],
    "related": {
        "users": [
            {
                "id": 42,
                "state": "pending",
                "name": "John Doe",
                "picture": "http://example.com/image/7g97g.jpg",
                "encryptionPubkey": "(base64-encoded data)"
            }
        ]
    },
    "meta": {}
}
```

#### Confirm a pending device

Sets the pending device's state to `active`.

```
PATCH /devices/:device_id
```
```json
{
    "state": "active"
}
```

Returns `204 No Content` on success.

Returns `404 Not Found` if the device with the given ID doesn't exist.

Returns `409 Conflict` if the device was not `pending`.

#### Block a device / Log out

Sets the device's state to `blocked`.

```
PATCH /devices/:device_id
```
```json
{
    "state": "blocked",
    "blockTime": "(RFC 3339 timestamp)"
}
```

Returns `204 No Content` on success.

Returns `404 Not Found` if the device with the given ID doesn't exist.

Returns `409 Conflict` if the device has already been blocked.

### WebSocket connection management

#### Make WebSocket URL

```
POST /ws_urls
```

##### Response

Returns `200 OK` on success:

```json
{
    "socketUrl": "wss://xyz"
}
```

Returns a single-use WebSocket URL which includes authentication information. Use it to connect to the WebSocket API. Expires if not used within 1 minute.


### Conversations

Conversations encompass channels and private group or 1:1 messages.

#### Get conversations

```
GET /conversations
```

##### Response

Returns `200 OK` on success:

`id` is a number >= 1;
`type` is one of "channel", "group";
`title` is a string (non-empty for type channel);
`participantIds` is a list of IDs of participants in this conversation;

`related.permissions` contains the permissions for the conversations in `objects` with the following fields:

`conversationId` is the related conversation's ID;
`conversationKeyId` is the ID of the symmetric encryption key;
`conversationKey` is the symmetric encryption key;
`ownerId` is the user who gets the permission;
`creatorId` is the user who created the permission;
`validFrom` timestamp (RFC 3339) from which this permission's key should be used;
`signature` TODO.

The server should filter the list of permissions for the authenticated user.

```json
{
    "objects": [
        {
            "id": "3eca5a5226a54134890bd6a648b54c04",
            "type": "channel",
            "title": "Off topic",
            "participantIds": [1, 2, 3]
        }
    ],
    "related": {
        "permissions": [
            {
                "conversationId": "3eca5a5226a54134890bd6a648b54c04",
                "conversationKeyId": "961e57c49ac08a897349d862ccc3f2f2",
                "conversationKey": "(encrypted for user 2, base64 encoded)",
                "ownerId": 2,
                "creatorId": 1,
                "validFrom": "2018-01-01T11:11:11Z",
                "signature": "(signing device ID),(base64 encoded signature)"
            },
            {
                "conversationId": "3eca5a5226a54134890bd6a648b54c04",
                "conversationKeyId": "ef0a99b55a599f09e4f8663ee15864ac",
                "conversationKey": "(encrypted for user 2, base64 encoded)",
                "ownerId": 2,
                "creatorId": 1,
                "validFrom": "2018-02-01T11:11:11Z",
                "signature": "(signing device ID),(base64 encoded signature)"
            }
        ]
    },
    "meta": {}
}
```

#### Create a conversation

```
POST /conversations
```
```json
{
    "conversation": {
        "id": "97c6cd24be847d9dfa26ecfc1f21619b",
        "type": "channel",
        "title": "New channel",
        "participantIds": [1, 2]
    },
    "permissions": [
        {
            "conversationId": "97c6cd24be847d9dfa26ecfc1f21619b",
            "conversationKeyId": "961e57c49ac08a897349d862ccc3f2f2",
            "conversationKey": "(encrypted for user 2, base64 encoded)",
            "ownerId": 1,
            "creatorId": 1,
            "validFrom": "2018-03-01T11:11:11Z",
            "signature": "(signing device ID),(base64 encoded signature)"
        },
        {
            "conversationId": "97c6cd24be847d9dfa26ecfc1f21619b",
            "conversationKeyId": "961e57c49ac08a897349d862ccc3f2f2",
            "conversationKey": "(encrypted for user 2, base64 encoded)",
            "ownerId": 2,
            "creatorId": 1,
            "validFrom": "2018-03-01T11:11:11Z",
            "signature": "(signing device ID),(base64 encoded signature)"
        }
    ]
}
```

##### Response

Returns `204 No Content` on success.

Returns `409 Conflict` if

* a group conversation with the same participants or
* a channel with the same title or
* a permission with the same `conversationKeyId`

already exists.

#### Add permissions to a conversation

This happens when a user is invited to an existing channel conversation
and when a user rotates the channel key.

This is a bulk action because in case of key rotation, many permissions must be
sent at once.

```
POST /conversations/:conversation_id/permissions
```
```json
[
    {
        "conversationId": "594dcc35a0d120bb17c78371071276c7",
        "conversationKeyId": "961e57c49ac08a897349d862ccc3f2f2",
        "conversationKey": "(encrypted for user 1, base64 encoded)",
        "ownerId": 1,
        "creatorId": 1,
        "validFrom": "2018-03-01T11:11:11Z",
        "signature": "(signing device ID),(base64 encoded signature)"
    },
    {
        "conversationId": "594dcc35a0d120bb17c78371071276c7",
        "conversationKeyId": "961e57c49ac08a897349d862ccc3f2f2",
        "conversationKey": "(encrypted for user 2, base64 encoded)",
        "ownerId": 2,
        "creatorId": 1,
        "validFrom": "2018-03-01T11:11:11Z",
        "signature": "(signing device ID),(base64 encoded signature)"
    },
    {
        "conversationId": "594dcc35a0d120bb17c78371071276c7",
        "conversationKeyId": "961e57c49ac08a897349d862ccc3f2f2",
        "conversationKey": "(encrypted for user 3, base64 encoded)",
        "ownerId": 3,
        "creatorId": 1,
        "validFrom": "2018-03-01T11:11:11Z",
        "signature": "(signing device ID),(base64 encoded signature)"
    }
]
```

#### Get messages

```
GET /conversations/:conversation_id/messages?cursor=1234&limit=10
```

* `cursor` is optional. By default, the latest messages are returned.
* `limit` is optional. By default, a sensible number of messages is returned.

##### Response

Returns `200 OK` on success:

```json
{
    "objects": [
        {
            "id": 21,
            "timeSent": "(RFC 3339 timestamp)",
            "revision": 0,
            "context": {
                "version": 1,
                "parentMessageId": 9,
                "previousMessageId": 10,
                "conversationKeyId": "961e57c49ac08a897349d862ccc3f2f2",
                "deviceKeyId": "64a29fbb8301116e6c6366d78818d51a"
            },
            "encryptedMessage": "(base64-encoded data)" // or null iff deleted
        },
        {
            "id": 18
            // ...
        }
    ],
    "meta": {
        "nextCursor": "2345"
    }
}
```

Returns `404 Not Found` if there is no conversation with the given ID.

## WebSockets

The WebSocket API is the preferred means of communication with the Kullo Chat server.

### Events

Used by the server to notify clients of changes.

```json
{
    "type": "...",
    "meta": { },
    "data": { }
}
```

`type` contains the type of the event. `meta` and `data` can be used to send additional event-specific data.

#### Conversation added or updated

On conversation creation, update (name, members, unreads), deletion

```json
{
    "type": "conversation.updated", // or "conversation.added" when conversation was added
    "data": {
        "id": 333,
        "type": "channel",
        "title": "Off topic",
        "participantIds": [1, 2, 3]
    }
}
```

#### Conversation permission added

On conversation permission creation; sent to the owner

```json
{
    "type": "conversation_permission.added",
    "data": {
        "conversationId": "3eca5a5226a54134890bd6a648b54c04",
        "conversationKeyId": "56bf13d79e3dc0767d0f47a74f705d25",
        "conversationKey": "(encrypted for user 2, base64 encoded)",
        "ownerId": 2,
        "creatorId": 1,
        "validFrom": "2018-01-01Z",
        "signature": "(signing device ID),(base64 encoded signature)"
    }
}
```

#### Message added or updated

On new/edited message (deletion is an edit):

```json
{
    "type": "message.added", // or "message.updated" when a message was updated
    "data": {
        "id": 21,
        "timeSent": "(RFC 3339 timestamp)",
        "revision": 0,
        "context": {
            "version": 1,
            "parentMessageId": 9,
            "previousMessageId": 10,
            "conversationKeyId": "961e57c49ac08a897349d862ccc3f2f2",
            "deviceKeyId": "64a29fbb8301116e6c6366d78818d51a"
        },
        "encryptedMessage": "(base64-encoded data)" // or null iff deleted
    }
}
```

#### Reaction event

On new/removed reaction

#### Typing indicator event

On started/stopped typing

#### Presence event

On presence change (online/offline)


### Requests

Used by the clients to send changes to the server.

```json
{
    "type": "...",
    "id": 42,
    "data": { }
}
```

The ID must be unique per connection and is referenced in server response events:

```json
{
    "type": "response",
    "meta": {
        "requestId": 42,
        "error": null
    },
    "data": { }
}
```

#### Create/update/delete conversation

#### Join/leave a conversation

```json
{
    "type": "conversation.join", // or "conversation.leave" to leave a conversation
    "id": 42,
    "data": {
        "id": 333
    }
}
```

##### Response

```json
{
    "type": "response",
    "meta": {
        "requestId": 42,
        "error": null
    },
    "data": {
        "id": 333,
        "type": "channel",
        "title": "Off topic",
        "participantIds": [1, 2, 3]
    }
}
```

#### Post/update message

```json
{
    "type": "message.add", // or "message.update" to update a message
    "id": 42,
    "data": {
        "id": 1, // only included when type == "message.update"
        "context": {
            "version": 1,
            "parentMessageId": 9,
            "previousMessageId": 10,
            "conversationKeyId": "961e57c49ac08a897349d862ccc3f2f2",
            "deviceKeyId": "64a29fbb8301116e6c6366d78818d51a"
        },
        "encryptedMessage": "(base64-encoded data)" // or null iff deleted
    }
}
```

##### Response

```json
{
    "type": "response",
    "meta": {
        "requestId": 42,
        "error": null
    },
    "data": {
        "id": 21,
        "timeSent": "(RFC 3339 timestamp)",
        "revision": 0
    }
}
```

#### Request attachment upload URLs

```json
{
    "type": "attachments.add",
    "id": 333,
    "data": {
        "count": 1
    }
}
```

##### Response

```json
{
    "type": "response",
    "meta": {
        "requestId": 333,
        "error": null
    },
    "data": [
        {
            "id": "3df8g9z",
            "uploadUrl": "http://localhost:8000/blob_storage/3df8g9z"
        }
    ]
}
```

#### Get a device

This is usually done to retrieve a signature key (device pubkey).

```json
{
    "type": "device.get",
    "id": 333,
    "data": {
        "id": "(device ID as requested)"
    }
}
```

##### Response

```json
{
    "type": "response",
    "meta": {
        "requestId": 333,
        "error": null
    },
    "data": {
        "id": "(device ID as requested)",
        "ownerId": 42,
        "idOwnerIdSignature": "(device ID as requested),(base64-encoded data)",
        "pubkey": "(base64-encoded data)",
        "state": "active",
        "blockTime": null
    }
}
```

#### Get a user

```json
{
    "type": "user.get",
    "id": 333,
    "data": {
        "id": 22
    }
}
```

##### Response

```json
{
    "type": "response",
    "meta": {
        "requestId": 333,
        "error": null
    },
    "data": {
        "id": 22,
        "state": "active",
        "name": "John Doe",
        "picture": "http://example.com/image/7g97g.jpg",
        "encryptionPubkey": "(base64-encoded data)"
    }
}
```

#### Get a conversation permission

This is necessary when another user sends a message in a conversation
that is encrypted using a new conversation key (after key rotation).

This request requires `ownerId` to match the current user.
The pair `ownerId`, `conversationKeyId` uniquely identifies a conversation permission.

```json
{
    "type": "conversation_permission.get",
    "id": 333,
    "data": {
        "conversationKeyId": "56bf13d79e3dc0767d0f47a74f705d25"
    }
}
```

##### Response

```json
{
    "type": "response",
    "meta": {
        "requestId": 333,
        "error": null
    },
    "data": {
        "conversationId": "3eca5a5226a54134890bd6a648b54c04",
        "conversationKeyId": "56bf13d79e3dc0767d0f47a74f705d25",
        "conversationKey": "(encrypted for user 2, base64 encoded)",
        "ownerId": 2,
        "creatorId": 1,
        "validFrom": "2018-01-01Z",
        "signature": "(signing device ID),(base64 encoded signature)"
    },
}
```

#### Post/update reaction

#### Update typing indicator

#### Update presence

