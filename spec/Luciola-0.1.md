# Luciola Protocol v0.1

Version 0.1 · 2026-10-03

## 1. Overview

![lociola](../brand/luciola.png)

Luciola is a decentralized social protocol. It consists of two elements and one role: an **Identity** signs **Actions**; actions are stored on **Nodes** and flow between nodes along relationships.

### 1.1 Principles

1. All nodes are equal. A node's size does not affect its standing in the protocol. The protocol has no central node, no global index, and no shared blocklist.
2. Each node operator is responsible for their own node, including admission, blocking, storage, and bandwidth.
3. Content flows only along relationships. Without a relationship, content is never pushed to another node.
4. Identity belongs to the user. It is rooted in a master key held by the user and cannot be taken by any node.
5. The authoritative copy of every action lives on the node its author was on when signing it.

### 1.2 Requirements language

The key words MUST, MUST NOT, SHOULD, and MAY in this document are to be interpreted as described in RFC 2119.

## 2. Terminology

| Term | Definition |
| --- | --- |
| Identity | A user, uniquely determined by their master public key |
| Master key | The root key held by the user; used only to delegate, revoke, and migrate |
| Subkey | A key delegated by the master key to sign other actions on the identity's behalf |
| Node | A server that hosts identities and stores and serves actions, identified by a domain name |
| Operator | The person who runs a node |
| Home node | The node that an identity's current valid delegations point to |
| Origin node | The node named in an action's `node` field |
| Action | A record signed by an identity; the only unit of content in the protocol |
| Follower | An identity that follows another identity |
| Pending reply | A reply from another node that has been received but not yet approved |
| Approval | The act of allowing a pending reply to be shown |
| Placeholder | A marker shown in place of content under the visibility rules, pointing to its origin node |
| Address | An identity locator of the form `@handle@node` |

## 3. Encoding

- **Data format**: all actions and API payloads are UTF-8 JSON.
- **Canonicalization**: before hashing or signing, JSON MUST be canonicalized per RFC 8785 (JSON Canonicalization Scheme, JCS).
- **Binary values**: public keys, signatures, and hashes are encoded as base64url (RFC 4648 section 5, no padding) with an algorithm prefix: public keys as `ed25519:<base64url>`, hashes as `sha256:<base64url>`, signatures as `ed25519:<base64url>`.
- **Time**: always UTC in RFC 3339 format, e.g. `2026-10-03T08:00:00Z`.
- **Domains**: a node is identified by its fully qualified domain name in lowercase, without scheme or path.
- **Version**: this document defines version `0.1`. Every action MUST carry an `luciola` field with the value `"0.1"`; the HTTP API lives under the path prefix `/luciola/v0/`.

## 4. Identity and keys

An identity is determined by its master key. The master key authorizes subkeys through delegation, and subkeys sign everyday actions.

### 4.1 Algorithm

Both master keys and subkeys use Ed25519 (RFC 8032).

### 4.2 Master key

- The identity identifier is the master public key, of the form `ed25519:<base64url>`.
- The master key is generated and held by the user. It is used only to sign `delegate`, `revoke`, and `migrate` actions.
- Export format: the master key's 32-byte seed MUST be exportable and importable as a BIP39 mnemonic (24 English words, using the seed as entropy). All nodes and clients MUST support importing this format.
- If the master key is lost or leaked, the identity is lost. The protocol provides no recovery mechanism.

### 4.3 Subkeys and delegation

- A subkey is a separate Ed25519 key. Once bound to a node by a `delegate` action, it may sign all actions on the identity's behalf except those listed in 4.2.
- A subkey may be held by the node, which signs on the user's behalf, or by the user on their own device.
- A subkey is not an identity. Every action it signs belongs to the identity that delegated it.
- All valid delegations of an identity MUST point to the same node, which is the home node. A `delegate` pointing to a different node is invalid unless a `migrate` to that node precedes it.

### 4.4 Key state

- **Revocation**: `revoke` invalidates the named subkey. After a node receives a `revoke`, it MUST reject any action signed by that subkey that it receives for the first time afterwards, regardless of its `created` field. Actions already received and stored are unaffected.
- **Retirement**: `migrate` retires all delegations pointing to the old node. A retired subkey MUST NOT sign new actions: after a node receives a `migrate`, it MUST reject any action signed by a retired subkey that it receives for the first time afterwards and whose `created` is later than that `migrate`. Actions signed by a retired subkey whose `created` is earlier than the `migrate` remain valid, so that historical content on the old node can still be verified.
- A user who does not trust the old node MAY issue a `revoke` for the old subkey, in which case the revocation rule applies.

## 5. Actions

An action is the only unit of content in the protocol. Once signed, an action cannot be changed; the protocol provides no editing.

### 5.1 Structure

```json
{
  "luciola": "0.1",
  "id": "sha256:...",
  "type": "post",
  "author": "ed25519:...",
  "key": "ed25519:...",
  "node": "example.org",
  "created": "2026-10-03T08:00:00Z",
  "target": [],
  "content": {},
  "sig": "ed25519:..."
}
```

| Field | Description |
| --- | --- |
| `luciola` | Protocol version, always `"0.1"` |
| `id` | Unique identifier of the action, computed per 5.2 |
| `type` | Action type, see 5.4 |
| `author` | The author's identity, i.e. the master public key |
| `key` | The public key that signed the action; equals `author` when signed by the master key |
| `node` | The node the author was on when signing, i.e. the origin node |
| `created` | Creation time declared by the author; used only for display, ordering, and the retirement rule in 4.4 |
| `target` | Array of referenced action ids; empty when there is no reference |
| `content` | Type-specific content object, see 5.5 |
| `sig` | Signature over `id` |

All fields are required. An action MUST NOT contain fields other than those listed above.

### 5.2 Identifier and signature

1. Take all fields except `id` and `sig` and canonicalize them with JCS.
2. Compute SHA-256 over the canonical form and encode it as `sha256:<base64url>`; this is the `id`.
3. Sign the UTF-8 bytes of the `id` string with the private key matching `key`, and encode the result as `ed25519:<base64url>`; this is the `sig`.

### 5.3 Validation

A node receiving an action MUST check the following in order and reject the action if any check fails:

1. All fields are present and well-typed, and `luciola` is `"0.1"`.
2. The `id` recomputed per 5.2 matches the `id` field.
3. `sig` verifies under `key`.
4. If `type` is `delegate`, `revoke`, or `migrate`, `key` MUST equal `author`. Otherwise, `key` MUST have a valid delegation signed by `author` that is neither revoked nor retired under 4.4.
5. `content` and `target` conform to 5.5 for that type.

### 5.4 Types

| Type | Signed by | `target` | Meaning |
| --- | --- | --- | --- |
| `post` | Subkey | Empty | A post |
| `reply` | Subkey | [the replied-to action] | A reply to a post or reply |
| `forward` | Subkey | [the forwarded action] | A forward |
| `approve` | Subkey | [the approved reply] | An approval |
| `follow` | Subkey | Empty | Follow |
| `unfollow` | Subkey | Empty | Unfollow |
| `remove` | Subkey | Empty | Remove a follower |
| `block` | Subkey | Empty | Block |
| `unblock` | Subkey | Empty | Unblock |
| `delete` | Subkey | [the deleted action] | Delete one's own action |
| `profile` | Subkey | Empty | Profile |
| `delegate` | Master key | Empty | Delegate a subkey |
| `revoke` | Master key | Empty | Revoke a subkey |
| `migrate` | Master key | Empty | Migrate |

### 5.5 Content

- `post`: `{ "text": string, "media": array of media, "mentions": array of identities }`. `text` is plain text of at most 10,000 characters; `media` has at most 4 entries, formatted per section 13; `mentions` lists the identities mentioned in the text. All three fields MUST be present and MAY be empty.
- `reply`: the fields of `post`, plus `"root"`, the id of the root post of the thread.
- `forward`, `approve`, `delete`: `{}`. The signer of `approve` MUST be the author of the action the approved reply responds to; the signer of `delete` MUST be the author of the deleted action.
- `follow`, `unfollow`, `remove`, `block`, `unblock`: `{ "subject": identity }`.
- `profile`: `{ "handle": string, "name": string, "bio": string, "avatar": media or null }`. `handle` follows section 12; `name` is at most 64 characters; `bio` is at most 1,000 characters. An identity's profile is the one with the latest `created`.
- `delegate`: `{ "subkey": public key, "node": domain }`.
- `revoke`: `{ "subkey": public key }`.
- `migrate`: `{ "node": domain of the new node }`.

### 5.6 Visibility scope

All actions in the protocol are public. A node MAY offer users content visible only to themselves; such content is not an action, MUST NOT be pushed, and MUST NOT appear in any API response.

## 6. Nodes

A node hosts identities, stores actions, and pushes and serves them according to the rules. Signature verification does not depend on any node; anyone holding the public key can verify.

- **Admission**: the operator decides which identities may register on or import into the node.
- **Same-node trust**: replies between identities on the same node need no approval.
- **Identity index**: a node MUST maintain a handle index of its identities for address resolution (section 12).
- **Default execution**: for `delete` and `migrate`, a node executes the rules in this document by default; the operator MAY choose not to, and bears the consequences.
- **Data export**: whether a node offers users a packaged export of their data is up to the operator.
- **Cost**: a node MUST be able to receive and store all content within the limits set by this document. Storage and bandwidth are borne by the operator.

## 7. Flow

While both sides are online, new actions arrive by push. Completeness comes from pulling from origin nodes. Content a node missed while offline is filled in by pulling once it is back.

### 7.1 Push recipients

After an action is signed, the author's home node pushes it to the nodes in the table below. Each node receives it once.

| Type | Recipients |
| --- | --- |
| `post`, `profile` | Nodes of the author's followers; home nodes of identities in `mentions` |
| `reply` | Origin node of the replied-to action; nodes of the author's followers; home nodes of identities in `mentions` |
| `forward` | Nodes of the author's followers; origin node of the forwarded action |
| `approve` | Origin node of the approved reply; nodes of the author's followers |
| `follow`, `unfollow`, `remove` | Home node of `subject` |
| `delete` | The same recipients as the deleted action |
| `delegate`, `revoke` | Nodes of the author's followers |
| `migrate` | Nodes of the author's followers; the new node also sends it to the old home node (section 11) |
| `block`, `unblock` | Not pushed |

### 7.2 Push rules

- Pushes use `POST /luciola/v0/inbox` (section 14).
- Each action is pushed to each recipient node once. A 2xx response means delivered.
- If a recipient node is unreachable, the sender MUST drop the push and MUST NOT retry.
- The recipient MUST validate each action per 5.3 and store it once valid.

### 7.3 Block reports

`block` takes effect only on the blocker's node. When a block happens, the blocker's node SHOULD send a report to the blocked identity's home node (`POST /luciola/v0/reports`). The report contains only the blocked identity and the ids of related actions, and MUST NOT contain any information about the blocker. A report is a signal; whether the receiving node acts on it is up to its operator.

### 7.4 Pull

- To show an action, an identity's history, or a post's thread, a node pulls from the relevant origin nodes.
- The authoritative copy of an action is served by its origin node. If the origin node is unreachable, unrelated content cannot be fetched.

### 7.5 Catch-up

- A node MUST record the last time it was online.
- When it comes back online, it MUST request, from the home node of every identity followed by its identities, the actions after that time (the `since` parameter), and store them.
- If a home node is unreachable at that moment, it is skipped and filled in later, when it becomes reachable or a user visits that identity.

## 8. Replies and approval

Replies from other nodes are hidden by default; the party being replied to decides whether they are shown.

### 8.1 When approval is needed

When the origin node of the replied-to action receives a `reply`:

- If the replier and the replied-to author are on the same node, the reply is shown.
- If the replied-to author follows the replier, the reply is shown.
- Otherwise, the reply becomes a pending reply.

### 8.2 Approval

- The right to approve belongs to the author of the replied-to action.
- Approval applies to a single reply and takes one of two forms: signing an `approve`, or the replied-to author signing a `reply` to the pending reply. Such a reply counts as approval of it, and all nodes MUST treat it as such.
- After a reply is approved, later replies from the same replier are again judged by 8.1.

### 8.3 Pending replies

- For a given replied-to action, an identity may have only one pending reply before being approved. The receiving node MUST reject a second one with 409.
- A node SHOULD notify the replied-to author of the first pending reply from an identity. Until the author acts on any pending reply from that identity, further pending replies from it are not notified.
- Pending replies do not expire. The node keeps them until they are approved, cleared by a block, or deleted by their author.

### 8.4 Blocking

- When the replied-to author blocks an identity, their node MUST discard all pending replies from that identity and reject its later replies.
- The node sends a block report per 7.3.

### 8.5 Approval within a thread

Within a thread, each reply is approved by the party it replies to, on that party's own node. For example: a posts, b's reply is approved, c replies to b; whether c's reply is approved is b's decision.

## 9. Visibility

What is shown depends on where the viewer enters. This section defines how a node presents content to users.

### 9.1 Viewing on one's own node

When a user views a thread, a forward, or a profile on their own node:

1. Actions by identities the user follows are shown.
2. Actions by identities the user does not follow, but that have been approved by any identity, are shown as placeholders. A placeholder MUST point to the action's origin node, where the user can view it.
3. Pending replies not approved by any identity are neither shown nor represented by a placeholder.
4. Actions by identities on the same node are shown.

### 9.2 Entering through another node

A user who enters through a link to another node sees that node's view, governed by the approvals on that node.

### 9.3 Deleted root

When the root post of a thread is deleted, the replies under it remain on their own origin nodes, and the root's position shows "original post deleted".

## 10. Caching and deletion

### 10.1 Caching

A node MUST store the actions, and their media, from other nodes that are related to its own identities, including:

- actions by identities its identities follow;
- actions its identities have replied to, forwarded, or approved;
- actions in threads its identities take part in.

Cached actions keep their original signatures and can be verified by any node. When an origin node is unreachable, the node MUST keep showing this content from its cache. Unrelated content is not cached.

### 10.2 Deletion

- An author MAY sign a `delete` to delete their own action. A `delete` itself cannot be deleted.
- A node receiving a `delete` by default deletes the referenced action and any media used only by it; the operator MAY choose not to.
- The origin node of a deleted action MUST return 410 for its id.
- Replies under a deleted post are not deleted with it; 9.3 applies.

## 11. Migration

Migration moves the identity, not its history or relationships.

### 11.1 Procedure

1. The user imports the master key on the new node (the BIP39 format of 4.2) and obtains a handle there.
2. The user signs a `migrate` with the master key (`node` set to the new node), then a `delegate` pointing to the new node, and then a new `profile` with the new subkey.
3. The new node sends the `migrate` to the inbox of the old home node and pushes it per 7.1.
4. On receiving the `migrate`, the old home node by default pushes it to the nodes of the identity's followers; the operator MAY choose not to.
5. A follower's node receiving the `migrate` MUST show the migration notice to the follower and MUST NOT transfer the follow relationship automatically. Whether to follow the new address is up to the follower, by signing a `follow`.

### 11.2 What moves

- Moves with the identity: the master key, the handle (obtained again on the new node), and the profile.
- Does not move: existing actions, the follower list, and the following list. Existing actions stay on the old node and remain verifiable under 4.4.
- If the handle is already taken on the new node, the user MUST choose another handle or another node.
- The protocol does not require the old node to point to the new node. Resolving the identity's old address on the old node returns 410, without the new address.

## 12. Address resolution

An address has the form `@handle@node`. The protocol provides no search by name or content; an identity is found by obtaining its address.

### 12.1 Handle

- Consists of lowercase letters a–z, digits 0–9, and underscores, 1 to 30 characters long.
- Unique within a node, first come, first served.

### 12.2 Resolution

```
GET https://{node}/.well-known/luciola?handle={handle}
```

Response:

```json
{ "author": "ed25519:...", "handle": "...", "node": "..." }
```

- The resolver MUST fetch the identity's latest `profile` and check that its `handle` and `node` match the response; otherwise resolution fails.
- An unknown handle returns 404; an identity that has migrated away returns 410.

### 12.3 Access from one's own node

A user enters an address on their own node. That node resolves it, pulls the other identity's profile and actions, and shows them locally. Following, replying, and other operations all happen on the user's own node.

## 13. Media

Media is not embedded in actions; actions reference it by hash. The hash is signed along with the action, so any copy can be checked.

### 13.1 Reference format

```json
{ "hash": "sha256:...", "type": "image/webp", "size": 345678, "alt": "image description" }
```

- `hash`: SHA-256 of the media file's raw bytes.
- `type`: media type, one of the values in 13.2.
- `size`: size in bytes.
- `alt`: alternative text, MAY be an empty string, at most 1,000 characters.

### 13.2 Formats and limits

| Kind | Allowed types | Per-file limit |
| --- | --- | --- |
| Image | `image/jpeg`, `image/png`, `image/webp`, `image/gif` | 10 MB |
| Video | `video/mp4` (H.264 video, AAC audio) | 100 MB |

Each action references at most 4 media items. 1 MB is 1,048,576 bytes.

### 13.3 Storage and retrieval

- The original file is served by the action's origin node at `GET /luciola/v0/media/{hash}`.
- Related nodes keep copies per 10.1.
- A fetcher MUST check that the file's hash, type, and size match the reference, and discard it if they do not.

## 14. HTTP API

Nodes communicate only over HTTPS. Request and response bodies are JSON, except for media. There is no transport-level authentication between nodes; trust comes entirely from action signatures.

### 14.1 General rules

- Ids, identities, and hashes in path parameters use the full prefixed string, e.g. `/luciola/v0/actions/sha256:...`.
- List endpoints use cursor pagination: `limit` defaults to 50, maximum 100; `cursor` takes the `next` value from the previous page; `next` is `null` when there are no more pages.
- A node MAY rate-limit requests and returns 429 when the limit is exceeded.

### 14.2 Endpoints

| Method | Path | Purpose |
| --- | --- | --- |
| POST | `/luciola/v0/inbox` | Receive pushes |
| GET | `/luciola/v0/actions/{id}` | Get a single action |
| GET | `/luciola/v0/actions/{id}/replies` | Get the reply index of an action |
| GET | `/luciola/v0/identities/{author}/actions` | Get actions an identity signed on this node |
| GET | `/luciola/v0/identities/{author}/keys` | Get an identity's `delegate`, `revoke`, and `migrate` actions |
| GET | `/luciola/v0/identities/{author}/profile` | Get an identity's latest `profile` |
| POST | `/luciola/v0/reports` | Receive block reports |
| GET | `/luciola/v0/media/{hash}` | Get a media file |
| GET | `/.well-known/luciola?handle={handle}` | Address resolution (section 12) |

### 14.3 Receiving pushes

Request: `{ "actions": [action, ...] }`, at most 100 per request.

Response 200: `{ "results": [{ "id": "...", "code": 200 }, ...] }`, one result per action. `code` is 200 (accepted), 400 (validation failed), 409 (violates 8.3 or the sender is blocked), or 413 (exceeds a limit).

### 14.4 Fetching actions

- `GET /luciola/v0/actions/{id}`: returns the action itself; 404 if not found, 410 if deleted.
- `GET /luciola/v0/actions/{id}/replies`: returns `{ "replies": [{ "id": "...", "node": "..." }], "next": ... }`, containing only replies that are shown under this node's rules and no pending replies.
- `GET /luciola/v0/identities/{author}/actions?since={time}`: returns `{ "actions": [...], "next": ... }`, containing actions the identity signed on this node and that this node stored after `since`, in ascending order of storage time; excludes `block` and `unblock`.
- `GET /luciola/v0/identities/{author}/keys`: returns `{ "actions": [...] }` in ascending order of `created`.
- `GET /luciola/v0/identities/{author}/profile`: returns the latest `profile` action; 404 if the identity is not on this node, 410 if it has migrated away.

### 14.5 Block reports

Request: `{ "about": identity, "actions": [action id, ...] }`. Returns 202 on success.

### 14.6 Media

`GET /luciola/v0/media/{hash}` returns the file's raw bytes with `Content-Type` set to the type in the reference.

### 14.7 Status codes

| Code | Meaning |
| --- | --- |
| 200 | Success |
| 202 | Report accepted |
| 400 | Malformed request or invalid signature |
| 404 | Not found |
| 409 | Violates the pending-reply limit, or blocked |
| 410 | Deleted or migrated away |
| 413 | Exceeds a size limit |
| 429 | Too many requests |

## 15. Scope

This protocol does not define the following, and implementations MUST NOT provide them between nodes by extending action types or fields:

- editing a signed action;
- likes or any count-based feedback;
- visibility scopes other than public and private-to-self;
- search by name or content, or any global index;
- blocklists shared between nodes;
- an old node pointing or forwarding to a new node.
