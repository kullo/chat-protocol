Message format
==============

Everything described in this file is part of the encrypted message content,
so nothing in here affects the client/server interface.

## Main structure

```json
{
    "type": "text",
    "content": "Moie!",
    "attachments": []
}
```

`attachments` may be `null` or unset. In both cases, client must treat it like
an empty list.

### Types

* In type "text", the `content` field contains plain text
* In type "reaction", the `content` field contains an emoji

## Attachments

An array of attachment objects like

```json
{
    "id": "f8cb093bcbcefc48edc544a5853cc773",
    "name": "img2.png",
    "mimeType": "image/png",
    "encryption": {
        "algorithm": "chacha20poly1305-ietf-nonce12prefixed",
        "key": "B/lViSzth+5SybaOk6TvE6HGPdQMSCAQs1MMlg2S7AA="
    },
    "thumbnail": {
        "id": "cdd28827b7e17a5c1c83580fcdc6974b",
        "mimeType": "image/jpeg",
        "width": 200,
        "height": 100,
        "encryption": {
            "algorithm": "chacha20poly1305-ietf-nonce12prefixed",
            "key": "wa4WgcueHGJSZ20IACWfj8XVj6g5PWEBFch229l0BY4="
        }
    }
}
```

with

* `id` used to build the attachment download URL,
* `name` the file name,
* `mimeType` the file type,
* `conversationKeyId` the ID of the key used to encrypt this attachment,
* `thumbnail` a thumbnail for this attachment (can be null).
