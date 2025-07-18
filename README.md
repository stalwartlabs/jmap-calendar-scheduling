# JMAP for Calendar Scheduling

## 1. Introduction

This document defines an extension to the [JSON Meta Application Protocol (JMAP)](https://datatracker.ietf.org/doc/html/rfc8620) for supporting explicit calendar scheduling operations. This extension is identified by the capability string `urn:ietf:params:jmap:calendars:scheduling`.

JMAP for Calendars \[RFC XXX] includes basic support for calendar scheduling by allowing clients to send and receive scheduling messages through the manipulation of calendar events. However, the scheduling model defined in JMAP for Calendars is based on the implicit scheduling model also used in CalDAV Scheduling \[RFC6638]. In this model, scheduling actions are triggered automatically by the server in response to changes to calendar data, and clients have limited control over the handling of scheduling messages.

While implicit scheduling is suitable for many organizational use cases, it imposes several limitations that hinder flexibility, transparency, and user control. Notably:

* Clients have no control over which scheduling messages are applied to their calendars; servers are required to automatically apply incoming invitations received over iTIP \[RFC5546], including via iMIP \[RFC6047].
* This opens a potential vector for abuse, where external actors can deliver unsolicited or malicious invitations to a user's calendar, such as through spam messages delivered over iMIP.
* The full range of iTIP scheduling functionality is not available in the implicit model. For example, actions such as sending counters (`COUNTER`), declining a counter-proposal (`DECLINECOUNTER`), replacing the organizer (`REQUEST` from a new organizer), or selectively processing or rejecting scheduling messages are unsupported or inconsistently implemented.

This specification introduces *JMAP for Calendar Scheduling*, an extension that defines an explicit scheduling model for JMAP. This model allows clients to receive, review, and explicitly act on scheduling messages rather than relying solely on server-side automation. It introduces two new JMAP data types: `CalendarSchedulingMessage` which represents an incoming iTIP message received by the user (including iMIP-delivered messages), optionally pending client review or action. And `CalendarSchedulingRequest` which represents an outbound scheduling message initiated by the user or client, to be delivered to one or more recipients via the appropriate transport.

By supporting these new objects and related methods, this extension enables clients to:

* View, filter, and inspect incoming scheduling messages.
* Explicitly control whether and how an incoming scheduling message affects their calendar data.
* Compose and send a full range of outbound iTIP messages, including those not supported in the implicit model.

Servers implementing this extension may continue to support the implicit scheduling model defined in JMAP for Calendars. This allows clients to choose the model most appropriate for their use case, whether that be automatic handling for intra-organizational scheduling, or explicit handling for user-mediated control of external invitations.

The goal of this extension is to provide a coherent and secure framework for interoperable, client-driven calendar scheduling that aligns with the full capabilities of the iTIP protocol while preserving the usability and simplicity of JMAP for Calendars.

### 1.1 Addition to the Capabilities Object

The capabilities object is returned as part of the JMAP Session object; see Section 2 of [RFC8620]. This document defines one additional capability URI called `urn:ietf:params:jmap:calendars:scheduling`. This capability indicates that the server supports the JMAP for Calendar Scheduling extension.

The value of this property in an account's "accountCapabilities" property is an object that MUST contain the following information on server capabilities and permissions for that account:

- `maxSchedulingMessagesPerRequest`: `UnsignedInt`
    > The maximum number of scheduling messages that can be processed in a single request. This is a server-defined limit to prevent abuse or excessive load.
- `maxRecipientsPerMessage`: `UnsignedInt|null`
    > The maximum number of recipients that can be included in a single scheduling message. If `null`, there is no limit.
- `outboundScheduling`: `Boolean`
    > Indicates whether the server supports sending outbound scheduling messages. If `false`, clients can only receive and process incoming scheduling messages.

## 2. Scheduling Message

A `CalendarSchedulingMessage` object represents an incoming iTIP \[RFC5546] or iMIP \[RFC6047] message received by the user. These messages convey scheduling operations such as invitations, replies, cancellations, and updates. The type of operation is identified in the `method` property, which indicates the iTIP method used (e.g., `REQUEST`, `REPLY`, `CANCEL`).

The original iTIP message content is provided in the `message` property, expressed in JSCalendar format. Servers MUST convert iTIP messages from iCalendar format to JSCalendar using the transformation rules defined in \[RFC XXXX] (*JSCalendar: Converting from and to iCalendar*).

If the iTIP message refers to a calendar event that exists in the user's account, the `calendarEventId` property contains the identifier of that event. In such cases, the server MAY also include the `eventPatch` property, which is a patch object representing the changes that would be applied to the event if the scheduling message were processed. This allows clients to inspect and preview the proposed changes.

To support flexible workflows, clients can choose whether scheduling messages are processed automatically by the server or explicitly by the user. Servers MAY allow users to configure this behavior. When a scheduling message is automatically processed, the `processed` property is set to `true`, and the corresponding changes are reflected in the user's calendar. If `processed` is `false`, the message awaits user action and does not affect any calendar event until explicitly processed.

This object allows clients to inspect, filter, and act on incoming scheduling messages in a secure and controlled manner, enabling explicit scheduling workflows where appropriate.

A `CalendarSchedulingMessage` object represents an iTIP message with additional metadata for JMAP processing. This object has the following properties:

- `id`: `Id`
    > The id of the `CalendarSchedulingMessage`.

- `receivedAt`: `UTCDateTime`
    > The time this message was received.

- `sentBy`: `Person`
    > Who sent the iTIP message. The `Person` object is defined in JMAP for Calendars.

- `comment`: `String|null`
    > Comment sent along with the iTIP message.

- `calendarEventId`: `Id|null`
    > The id of the `CalendarEvent` that this message is about, or `null` for new events that have not yet been assigned an id.

- `processed`: `Boolean` 
    > `True` if the event has been automatically processed by the server. If false, the user must take action to process the event. Scheduling messages with `processed` set to `true` MUST include a valid `calendarEventId` property.

- `method`: `String`
    > The method of the scheduling message. One of `REQUEST`, `REPLY`, `CANCEL`, `ADD`, `COUNTER`, `DECLINECOUNTER`, `REFRESH`, `PUBLISH`.

- `message`: `JSCalendar Event`
    > The iTIP message in JSCalendar format.

- `eventPatch`: `PatchObject|null`
    > A patch encoding the change between the data in the event property, and the data in the iTIP message. Or null if the message is not an update.


## 2.1.`CalendarSchedulingMessage/set`

This is a standard "/set" method as described in Section 5.3 of [RFC8620], with the following extra argument:

- **targetCalendarId: Id**
  > This argument is required when processing a scheduling message that does not reference an existing calendar event (i.e., `calendarEventId` is `null`) and the method implies event creation (e.g., `REQUEST`). It specifies the calendar in which the new event should be created.

A scheduling message is explicitly processed by the client by setting its `processed` property to `true` using the standard JMAP `/set` method. This signals to the server that the client wishes to apply the changes described in the scheduling message to the relevant calendar event.

If the scheduling message has already been processed (i.e., its `processed` property is already `true`), the server MUST reject the request and return an `alreadyProcessed` error. If the server is unable to apply the changes described in the scheduling message (such as due to a semantic conflict, missing referenced data, or validation failure), it MUST reject the update and return a `cannotProcess` error.

When the scheduling message does not reference an existing calendar event (i.e., the `calendarEventId` property is `null`), and the `method` implies event creation (e.g., `REQUEST`, `PUBLISH`, or `ADD`), the client MUST specify the `targetCalendarId` argument in the same `/set` call. This argument identifies the calendar in which the event described in the scheduling message should be created. If `targetCalendarId` is required but omitted, the server MUST return `invalidArguments` error.

Example:

```javascript
            [[ "CalendarSchedulingMessage/set", {
                "accountId": "x",
                "targetCalendarId": "d",
                "update": {
                  "a": {
                    "id": "a",
                    "processed": true,
                  }
                }
             }, "0" ]]
```

### 2.2. `CalendarSchedulingMessage/get`

This is a standard "/get" method as described in Section 5.1 of [RFC8620].

### 2.3. `CalendarSchedulingMessage/query`

This is a standard "/query" method as described in Section 5.5 of [RFC8620].

A `FilterCondition` object has the following properties:

- `after`: `UTCDateTime|null`
    > Only return messages received after this date.
- `before`: `UTCDateTime|null`
    > Only return messages received before this date.
- `method`: `String|null`
    > Only return messages with this iTIP method. One of `REQUEST`, `REPLY`, `CANCEL`, `ADD`, `COUNTER`, `DECLINECOUNTER`, `REFRESH`, `PUBLISH`.
- `from`: `String|null`
    > Only return messages sent by this person.
- `processed`: `Boolean|null`
    > Only return messages that have been processed (`true`) or not processed (`false`).
- `hasCalendarEvent`: `Boolean|null`
    > Only return messages that have a `calendarEventId`.
- `calendarEventId`: `Id|null`
    > Only return messages that relate to this calendar event.


## 3. Scheduling Request

A `CalendarSchedulingRequest` object represents an outgoing iTIP \[RFC5546] scheduling message initiated by the user. It allows a client to explicitly compose and send scheduling operations such as event invitations (`REQUEST`), replies (`REPLY`), cancellations (`CANCEL`), and other iTIP methods to one or more recipients.

To construct a `CalendarSchedulingRequest`, the client MUST specify the `method` property, identifying the iTIP method being used, and the `to` property, listing the intended recipients. The contents of the scheduling message are expressed in JSCalendar format.

Clients have two options for supplying the JSCalendar data to be included in the iTIP message:

- **Manual construction**: The client constructs the entire JSCalendar representation of the iTIP message and provides it in the `message` property. This method is intended for advanced clients that require full control over the structure and contents of the outgoing message.

- **Patch-based generation**: The client provides only a `calendarEventId` referencing an existing event, and a `eventPatch` object describing the changes to be sent. The server then generates the iTIP message from the patch. This is a simplified mechanism for clients that do not wish to handle the complexities of building iTIP-compliant messages themselves.

Only one of `message` or `eventPatch` MAY be provided in a request. If both are included, the server MUST reject the request with an `invalidArguments` error.

Implementation of the `CalendarSchedulingRequest` data type and related methods is OPTIONAL. Servers are only required to implement `CalendarSchedulingMessage`. Servers indicate their support for sending scheduling messages by setting the `outboundScheduling` property to `true` in the JMAP capabilities object for this extension.

If a server does not support outbound scheduling (`outboundScheduling` set to `false`), clients MUST use the implicit scheduling model defined in JMAP for Calendars and send scheduling messages by setting the `sendSchedulingMessages` property to `true` in a `CalendarEvent/set` request.

Even if a server advertises support for outbound scheduling via `CalendarSchedulingRequest`, clients MAY continue to use the implicit scheduling model instead, depending on their design preferences or requirements.

A `CalendarSchedulingRequest` object represents an outgoing scheduling request message. This object has the following properties:

- `id`: `Id`
   > The id of the `CalendarSchedulingMessage`.

 - `createdAt`: `UTCDateTime`
    > The time this message was created.

- `to`: `Person[]`
    > Who the iTIP message is addressed to. The `Person` object is defined in JMAP for Calendars.

- `comment`: `String|null`
    > Comment sent along with the change by the user that made it. (e.g. `COMMENT` property in an iTIP message), if any.

- `calendarEventId`: `Id|null`
    > The id of the `CalendarEvent` that this message is about, or `null` for `PUBLISH` messages that do not relate to a specific event.

- `method`: `String`
    > The method of the scheduling message. One of `REQUEST`, `REPLY`, `CANCEL`, `ADD`, `COUNTER`, `DECLINECOUNTER`, `REFRESH`, `PUBLISH`.

- `message`: `JSCalendar Event|null`
    > The iTIP message in JSCalendar format or null if the `eventPatch` is provided instead.

- `eventPatch`: `PatchObject|null`
    > A patch object representing the changes to be sent in the iTIP message, can only be used when `calendarEventId` is set. This is used as an alternative to the `message` property for clients who require more control over the changes being sent.

- `deliveryStatus`: `String[DeliveryStatus]|null` (server-set)
    > This represents the delivery status for each of the iTIP message's external recipients, if known. The `DeliveryStatus` object is defined in RFC8621.

### 3.1. `CalendarSchedulingRequest/set`

This is a standard "/set" method as described in Section 5.3 of [RFC8620].

The `CalendarSchedulingRequest/set` method is used to send outbound iTIP \[RFC5546] messages, such as invitations (`REQUEST`), replies (`REPLY`), counter-proposals (`COUNTER`), cancellations (`CANCEL`), publications (`PUBLISH`), and other supported scheduling operations.

To send a new event invitation, the client sets the `method` property to `"REQUEST"`, provides the `calendarEventId` referencing the calendar event to invite participants to, and sets both `message` and `eventPatch` to `null`. The server will construct the iTIP message from the referenced event.

Replies and updates to existing events are sent by specifying the appropriate `method` value (e.g., `REPLY`, `CANCEL`, `COUNTER`, etc.) and including the iTIP message content using one of the mechanisms described in Section 3: either by supplying a full `message` or by providing a `eventPatch`.

To send a `PUBLISH` message, the client sets the `method` to `"PUBLISH"`, MUST set `calendarEventId` to `null`, and MUST provide the iTIP message content in the `message` property.

The server is responsible for validating and delivering the generated iTIP message to the specified recipients using the appropriate transport mechanism.

Example:

```javascript
[[ "CalendarSchedulingRequest/set", {
  "accountId": "a0x9",
  "create": {
    "123": {
      "method": "REQUEST",
      "calendarEventId": "event-123",
      "to": [{
        "name": "Attendee Name",
        "email": "attendee@example.com"
      }],
      "comment": "Please join our meeting",
    }
  }
}, "0" ]]
```

### 3.2. `CalendarSchedulingRequest/get`

This is a standard "/get" method as described in Section 5.1 of [RFC8620].

### 3.3. `CalendarSchedulingRequest/query`

This is a standard "/query" method as described in Section 5.5 of [RFC8620].

A `FilterCondition` object has the following properties:

- `after`: `UTCDateTime|null`
    > Only return messages sent after this date.
- `before`: `UTCDateTime|null`
    > Only return messages sent before this date.
- `method`: `String|null`
    > Only return messages with this iTIP method. One of `REQUEST`, `REPLY`, `CANCEL`, `ADD`, `COUNTER`, `DECLINECOUNTER`, `REFRESH`, `PUBLISH`.
- `to`: `String|null`
    > Only return messages sent to this person.
- `hasCalendarEvent`: `Boolean|null`
    > Only return messages that have a `calendarEventId`.
- `calendarEventId`: `Id|null`
    > Only return messages that relate to this calendar event.


## 4. Additional Calendar Object properties

This document also defines two new JMAP for Calendar properties in the Calendar object for use in JMAP for Calendar Scheduling. These properties allow clients to control whether and how the server automatically processes incoming iTIP scheduling messages for the calendar.


### 4.1. `implicitSchedulingCreate`

Type: `Boolean`

The `implicitSchedulingCreate` property determines whether new scheduling messages received via iTIP that do not reference an existing event (such as the `REQUEST` method) are automatically processed by the server and result in new events being added to the calendar. By default, this property is set to `false`, meaning such messages are not processed unless explicitly handled by the client.

 If set to `true`, the server will automatically process qualifying messages and add the corresponding events to the calendar. Only one calendar in the account may have `implicitSchedulingCreate` set to `true` at any given time. If a client attempts to enable this property on a second calendar while another already has it enabled, the server MUST reject the request with an `invalidArguments` error.

### 4.2. `implicitSchedulingUpdate`

Type: `Boolean`

The `implicitSchedulingUpdate` property controls whether the server should automatically apply changes from iTIP messages that reference existing events in the calendar. These may include methods such as `REPLY`, `CANCEL`, or `REQUEST` targeting a known UID. 

When this property is set to `true`, the server automatically updates the referenced calendar event with the changes described in the iTIP message. This property is also set to `false` by default, requiring the client to process such messages explicitly.

### 4.3. Disabling Explicit Scheduling

If both `implicitSchedulingCreate` and `implicitSchedulingUpdate` are set to `true`, the calendar operates in fully implicit scheduling mode, and the client disables all of the functionality provided by this extension. In this configuration, the server handles the automatic processing of all incoming iTIP messages relevant to this calendar. 

However, even in implicit mode, the server MAY still provide `CalendarSchedulingMessage` objects for received scheduling messages, allowing clients to inspect the original message for auditing, logging, or user interface purposes.
