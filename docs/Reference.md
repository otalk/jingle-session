# 1.0.x API Reference

The `jingle-session` module is intended to be used in conjunction with [Jingle.js](https://github.com/otalk/jingle.js).

- [`Session`](#session)
  - [`new Session(opts)`](#new-sessionopts)
  - [`Session` Options](#session-options)
  - [`Session` Properties](#session-properties)
  - [`Session` Methods](#session-methods)
    - [`session.process(action, data, cb)`](#sessionprocessaction-data-cb)
    - [`session.send(action, data)`](#sessionsendaction-data)
    - [`session.start()`](#sessionstart)
    - [`session.accept()`](#sessionaccept)
    - [`session.cancel()`](#sessioncancel)
    - [`session.decline()`](#sessiondecline)
    - [`session.end([reason], [silent])`](#sessionendreason-silent)
    - [`session.on<SessionAction>(data, cb)`](#sessiononsessionactiondata-cb)


## `Session`

The `Session` class is intended to be a common base for covering the general requirements of a Jingle session. Subclasses of `Session` are needed to actually do interesting things, such as media or file transfer.

### `new Session(opts)`

- `opts` - An object summarizing the information in the session initiate request, suitable for passing directly to a session class constructor:
    - `sid` - The ID for the session, as provided by the initiating peer.
    - `peer` - The JID for the initiating peer (may be either a `{String}` or [`{JID}`](https://github.com/otalk/xmpp-jid)).
    - `peerID` - An alternative to `peer`, which MUST be a `{String}` (derived from `peer`).
    - `initiator` - This will always be `false`, as we are the one receiving the initiation request.
    - `descriptionTypes` - An array of content description names.
    - `transportTypes` - An array of content transport names.

### `Session` Properties

- `sid` - A unique ID for the session.
- `peer` - The JID for the initiating peer (may be either a `{String}` or [`{JID}`](https://github.com/otalk/xmpp-jid)).
- `peerID` - An alternative to `peer`, which MUST be a `{String}` (derived from `peer`).
- `state` - Always one of:
    - `starting`
    - `pending`
    - `active`
    - `ended`
- `connectionState` - Always one of:
    - `starting`
    - `connecting`
    - `connected`
    - `disconnected`
    - `interrupted`
- `pendingAction` - The name of the action that has just been sent to the peer, so we can check for tie-breaking if the same action is requested from both parties at the same time.
- `pendingDescriptionTypes` - The list of content types gathered from a session initiate request, so that we can perform tie-breaking if another session initiation request is received for the same content types.
- `isInitiator` - This flag is `true` if the `initiator` field is `true` when creating the session.
- `starting` - This flag is `true` when `.state` equals `'starting'`.
- `pending` - This flag is `true` when `.state` equals `'pending'`.
- `active` - This flag is `true` when `.state` equals `'active'`.
- `ended` - This flag is `true` when `.state` equals `'ended'`.
- `connecting` - This flag is `true` when `.connectionState` equals `'connecting'`.
- `connected` - This flag is `true` when `.connectionState` equals `'connected'`.
- `disconnected` - This flag is `true` when `.connectionState` equals `'disconnected'`.
- `interrupted` - This flag is `true` when `.connectionState` equals `'interrupted'`.

### `Session` Methods

#### `session.process(action, data, cb)`
- `action` - The session action to apply to the session.
- `data` - The contents of the `jingle` field of a Jingle packet.
- `cb([err])` - callback for returning potential errors, and triggering processing for the next queued action.
    - `err` - An error object if the action could not be performed
        - `condition` - The general error condition
        - `jingleCondition` - A Jingle specific error condition, if applicable

Calling this method places the action and its associated data and callback into the session's processing queue.

Each action is processed sequentially, calling `cb()` to trigger sending an ack result to the peer, and then process the next action. If `cb()` is given an error object, an error response will be sent to the peer.

This method is not intended to be called directly by the user, but is a required method for any `Session` instance to be able to work with the session jingle.

#### `session.send(action, data)`

- `action` - The session action the peer should perform based on this request.
- `data` - Information to insert into the `jingle` section of the packet. The `sid` and `action` fields will be automatically set.

Emits a `send` event for a new Jingle packet:

```js
// Send a session terminate message directly:
session.send('session-terminate', {
    reason: {
        condition: 'gone'
    }
});

// emitted send event: {
//    to: 'otherpeer@theirdomain.example',
//    type: 'set',
//    jingle: {
//        sid: 'sid123',
//
//        // the provided action:
//        action: 'session-terminate',
//
//        // the provided data:
//        reason: {
//            condition: 'gone'
//        }
//    }
//}
```

#### `session.start()`

Initiate a the session request to the peer. Calling this method will move the session from the `starting` state to `pending`, and will trigger an `outgoing` event on the session jingle.

The session should have been added to the session jingle with [`.addSession()`](#jingleaddsessionsession) before calling `.start()`.

```js
var session = new MyCustomSession({
    peer: 'otheruser@theirdomain.example',
    initiator: true
});

jingle.addSession(session);
jingle.on('outgoing', function (sess) {
    console.log('Outgoing session:', sess.sid === session.sid);
});

session.start();
// -> Outgoing session: true
```

#### `session.accept()`

Moves the session to the `active` state, and sends a `session-accept` action.

For a `Session` instance, this actually calls `.end('unsupported-applications')`.

```js
jingle.on('incoming', function (session) {
    // Auto-accept an incoming session
    session.accept();
});
```

#### `session.cancel()`

This is a shortcut for calling `session.end('cancel')`.

Calling `session.cancel()` should only be done after initiating the session and before it is accepted by the peer. After that point, calling `session.end()` is more appropriate.

#### `session.decline()`

This is a shortcut for calling `session.end('decline')`.

Calling `session.decline()` should only be done after receiving a session initiation request and before (or rather, instead of) accepting the session.

#### `session.end([reason], [silent])`

- `reason` - Why the session is being ended. This may be either a `{String}` or an object:
    - `condition` - The name of the reason
    - `text` - A freeform description of the reason
    - `alternativeSession` - If the condition is `alternative-session`, this is the `sid` value for that session.
- `silent` - If `true`, the session terminate message will not be generated to be sent to the peer.

Once `.end()` is called, the session moves to the `ended` state.

The list of valid `reason` (or `reason.condition`) values:

- `alternative-session`
- `busy`
- `cancel`
- `connectivity-error`
- `decline`
- `expired`
- `failed-application`
- `failed-transport`
- `general-error`
- `gone`
- `incompatible-parameters`
- `media-error`
- `security-error`
- `success`
- `timeout`
- `unsupported-applications`
- `unsupported-transports`

See [XEP-0166: Jingle Section 7.4](http://xmpp.org/extensions/xep-0166.html#def-reason) for more information on when each reason condition should be used.

```js
// We succesfully used the session, and are now ending it:
session.end('success');

// Declining a session in favor of an existing one:
session.end({
    condition: 'alternative-session',
    alternativeSession: 'othersessionsid'
});
```

#### `session.on<SessionAction>(data, cb)`

- `data` - The `jingle` payload of a Jingle packet
- `cb` - Callback for sending an ack or error to the peer, and allow processing of the next action.

Each session action has a corresponding `onX(data, cb)` method, where `X` is the action name in CamelCase without dashes.

The full list of standard session actions:

- `content-accept`
- `content-add`
- `content-modify`
- `content-reject`
- `content-remove`
- `description-info`
- `security-info`
- `session-accept`
- `session-info`
- `session-initiate`
- `session-terminate`
- `transport-accept`
- `transport-info`
- `transport-reject`
- `transport-replace`

See [XEP-0166: Jingle Section 7.2](http://xmpp.org/extensions/xep-0166.html#def-action) for more information on when each action is used.
