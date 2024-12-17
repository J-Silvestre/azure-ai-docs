---
title: Azure OpenAI Service Realtime API Reference
titleSuffix: Azure OpenAI
description: Learn how to use the Realtime API to interact with the Azure OpenAI service in real-time.
manager: nitinme
ms.service: azure-ai-openai
ms.topic: conceptual
ms.date: 12/12/2024
author: eric-urban
ms.author: eur
recommendations: false
---

# Realtime API (Preview) reference

[!INCLUDE [Feature preview](includes/preview-feature.md)]

The Realtime API is a WebSocket-based API that allows you to interact with the Azure OpenAI service in real-time. 

The Realtime API (via `/realtime`) is built on [the WebSockets API](https://developer.mozilla.org/docs/Web/API/WebSockets_API) to facilitate fully asynchronous streaming communication between the end user and model. Device details like capturing and rendering audio data are outside the scope of the Realtime API. It should be used in the context of a trusted, intermediate service that manages both connections to end users and model endpoint connections. Don't use it directly from untrusted end user devices.

> [!TIP]
> To get started with the Realtime API, see the [quickstart](realtime-audio-quickstart.md) and [how-to guide](./how-to/realtime-audio.md).

## Connection

The Realtime API requires an existing Azure OpenAI resource endpoint in a supported region. The API is accessed via a secure WebSocket connection to the `/realtime` endpoint of your Azure OpenAI resource.

You can construct a full request URI by concatenating:

- The secure WebSocket (`wss://`) protocol
- Your Azure OpenAI resource endpoint hostname, for example, `my-aoai-resource.openai.azure.com`
- The `openai/realtime` API path
- An `api-version` query string parameter for a supported API version such as `2024-10-01-preview`
- A `deployment` query string parameter with the name of your `gpt-4o-realtime-preview` model deployment

The following example is a well-constructed `/realtime` request URI:

```http
wss://my-eastus2-openai-resource.openai.azure.com/openai/realtime?api-version=2024-10-01-preview&deployment=gpt-4o-realtime-preview-1001
```

## Authentication

To authenticate:
- **Microsoft Entra** (recommended): Use token-based authentication with the `/realtime` API for an Azure OpenAI Service resource with managed identity enabled. Apply a retrieved authentication token using a `Bearer` token with the `Authorization` header.
- **API key**: An `api-key` can be provided in one of two ways:
  - Using an `api-key` connection header on the prehandshake connection. This option isn't available in a browser environment.
  - Using an `api-key` query string parameter on the request URI. Query string parameters are encrypted when using https/wss.

## Client events

There are nine client events that can be sent from the client to the server:

| Event | Description |
|-------|-------------|
| [RealtimeClientEventConversationItemCreate](#realtimeclienteventconversationitemcreate) | Send this client event when adding an item to the conversation. |
| [RealtimeClientEventConversationItemDelete](#realtimeclienteventconversationitemdelete) | Send this client event when you want to remove any item from the conversation history. |
| [RealtimeClientEventConversationItemTruncate](#realtimeclienteventconversationitemtruncate) | Send this client event when you want to truncate a previous assistant message's audio. |
| [RealtimeClientEventInputAudioBufferAppend](#realtimeclienteventinputaudiobufferappend) | Send this client event to append audio bytes to the input audio buffer. |
| [RealtimeClientEventInputAudioBufferClear](#realtimeclienteventinputaudiobufferclear) | Send this client event to clear the audio bytes in the buffer. |
| [RealtimeClientEventInputAudioBufferCommit](#realtimeclienteventinputaudiobuffercommit) | Send this client event to commit audio bytes to a user message. |
| [RealtimeClientEventResponseCancel](#realtimeclienteventresponsecancel) | Send this client event to cancel an in-progress response. |
| [RealtimeClientEventResponseCreate](#realtimeclienteventresponsecreate) | Send this client event to trigger a response generation. |
| [RealtimeClientEventSessionUpdate](#realtimeclienteventsessionupdate) | Send this client event to update the session's default configuration. |

### RealtimeClientEventConversationItemCreate

The client `conversation.item.create` event is used to add a new item to the conversation's context, including messages, function calls, and function call responses. This event can be used to populate a history of the conversation and to add new items mid-stream. Currently this event can't populate assistant audio messages.

If successful, the server responds with a `conversation.item.created` event, otherwise an `error` event is sent.

#### Event structure

```json
{
  "type": "conversation.item.create",
  "previous_item_id": "<previous_item_id>"
}
```

#### Properties

| Field | Type | Description | 
|-------|------|-------------|
| type | string | The event type must be `conversation.item.create`. | 
| previous_item_id | string | The ID of the preceding item after which the new item is inserted. If not set, the new item is appended to the end of the conversation. If set, it allows an item to be inserted mid-conversation. If the ID can't be found, then an error is returned and the item isn't added. | 
| item | [RealtimeConversationRequestItem](#realtimeconversationrequestitem) | The item to add to the conversation. | 

### RealtimeClientEventConversationItemDelete

The client `conversation.item.delete` event is used to remove an item from the conversation history. 

The server responds with a `conversation.item.deleted` event, unless the item doesn't exist in the conversation history, in which case the server responds with an error.

#### Event structure

```json
{
  "type": "conversation.item.delete",
  "item_id": "<item_id>"
}
```

#### Properties

| Field | Type | Description | 
|-------|------|-------------| 
| type | string | The event type must be `conversation.item.delete`. |
| item_id | string | The ID of the item to delete. | 

### RealtimeClientEventConversationItemTruncate

The client `conversation.item.truncate` event is used to truncate a previous assistant message's audio. The server produces audio faster than realtime, so this event is useful when the user interrupts to truncate audio that was sent to the client but not yet played. The server's understanding of the audio with the client's playback is synchronized.

Truncating audio deletes the server-side text transcript to ensure there isn't text in the context that the user doesn't know about.

If the client event is successful, the server responds with a `conversation.item.truncated` event.

#### Event structure

```json
{
  "type": "conversation.item.truncate",
  "item_id": "<item_id>",
  "content_index": 0,
  "audio_end_ms": 0
}
```

#### Properties

| Field | Type | Description | 
|-------|------|-------------| 
| type | string | The event type must be `conversation.item.truncate`. | 
| item_id | string | The ID of the assistant message item to truncate. Only assistant message items can be truncated. | 
| content_index | integer | The index of the content part to truncate. Set this property to "0". | 
| audio_end_ms | integer | Inclusive duration up to which audio is truncated, in milliseconds. If the audio_end_ms is greater than the actual audio duration, the server responds with an error. | 

### RealtimeClientEventInputAudioBufferAppend

The client `input_audio_buffer.append` event is used to append audio bytes to the input audio buffer. The audio buffer is temporary storage you can write to and later commit. 

In Server VAD (Voice Activity Detection) mode, the audio buffer is used to detect speech and the server decides when to commit. When server VAD is disabled, the client can choose how much audio to place in each event up to a maximum of 15 MiB. For example, streaming smaller chunks from the client can allow the VAD to be more responsive. 

Unlike most other client events, the server doesn't send a confirmation response to client `input_audio_buffer.append` event.

#### Event structure

```json
{
  "type": "input_audio_buffer.append",
  "audio": "<audio>"
}
```

#### Properties

| Field | Type | Description | 
|-------|------|-------------| 
| type | string | The event type must be `input_audio_buffer.append`. | 
| audio | string | Base64-encoded audio bytes. This value must be in the format specified by the `input_audio_format` field in the session configuration. | 

### RealtimeClientEventInputAudioBufferClear

The client `input_audio_buffer.clear` event is used to clear the audio bytes in the buffer. 

The server responds with an `input_audio_buffer.cleared` event.

#### Event structure

```json
{
  "type": "input_audio_buffer.clear"
}
```

#### Properties

| Field | Type | Description | 
|-------|------|-------------| 
| type | string | The event type must be `input_audio_buffer.clear`. | 

### RealtimeClientEventInputAudioBufferCommit

The client `input_audio_buffer.commit` event is used to commit the user input audio buffer, which creates a new user message item in the conversation. Audio is transcribed if `input_audio_transcription` is configured for the session.
 
When in server VAD mode, the client doesn't need to send this event, the server commits the audio buffer automatically. Without server VAD, the client must commit the audio buffer to create a user message item. This client event produces an error if the input audio buffer is empty. 

Committing the input audio buffer doesn't create a response from the model. 

The server responds with an `input_audio_buffer.committed` event.

#### Event structure

```json
{
  "type": "input_audio_buffer.commit"
}
```

#### Properties

| Field | Type | Description | 
|-------|------|-------------| 
| type | string | The event type must be `input_audio_buffer.commit`. | 

### RealtimeClientEventResponseCancel

The client `response.cancel` event is used to cancel an in-progress response. 

The server responds with a `response.cancelled` event or an error if there's no response to cancel.

#### Event structure

```json
{
  "type": "response.cancel"
}
```

#### Properties

| Field | Type | Description | 
|-------|------|-------------| 
| type | string | The event type must be `response.cancel`. | 

### RealtimeClientEventResponseCreate

The client `response.create` event is used to instruct the server to create a response via model inferencing. When the session is configured in server VAD mode, the server creates responses automatically.

A response includes at least one `item`, and can have two, in which case the second is a function call. These items are appended to the conversation history.

The server responds with a [`response.created`](#realtimeservereventresponsecreated) event, one or more item and content events (such as `conversation.item.created` and `response.content_part.added`), and finally a [`response.done`](#realtimeservereventresponsedone) event to indicate the response is complete.

> [!NOTE]
> The client `response.create` event includes inference configuration like 
`instructions`, and `temperature`. These fields can override the session's configuration for this response only.

#### Event structure

```json
{
  "type": "response.create"
}
```

#### Properties

| Field | Type | Description | 
|-------|------|-------------| 
| type | string | The event type must be `response.create`. | 
| response | [RealtimeResponseOptions](#realtimeresponseoptions) | The response options. |

### RealtimeClientEventSessionUpdate

The client `session.update` event is used to update the session's default configuration. The client can send this event at any time to update the session configuration, and any field can be updated at any time, except for voice. 

Only fields that are present are updated. To clear a field (such as `instructions`), pass an empty string. 

The server responds with a `session.updated` event that contains the full effective configuration.

#### Event structure

```json
{
  "type": "session.update"
}
```

#### Properties

| Field | Type | Description | 
|-------|------|-------------| 
| type | string | The event type must be `session.update`. | 
| session | [RealtimeRequestSession](#realtimerequestsession) | The session configuration. |

## Server events

There are 28 server events that can be received from the server:

| Event | Description |
|-------|-------------|
| [RealtimeServerEventConversationCreated](#realtimeservereventconversationcreated) | Server event when a conversation is created. Emitted right after session creation. |
| [RealtimeServerEventConversationItemCreated](#realtimeservereventconversationitemcreated) | Server event when a conversation item is created. |
| [RealtimeServerEventConversationItemDeleted](#realtimeservereventconversationitemdeleted) | Server event when an item in the conversation is deleted. |
| [RealtimeServerEventConversationItemInputAudioTranscriptionCompleted](#realtimeservereventconversationiteminputaudiotranscriptioncompleted) | Server event when input audio transcription is enabled and a transcription succeeds. |
| [RealtimeServerEventConversationItemInputAudioTranscriptionFailed](#realtimeservereventconversationiteminputaudiotranscriptionfailed) | Server event when input audio transcription is configured, and a transcription request for a user message failed. |
| [RealtimeServerEventConversationItemTruncated](#realtimeservereventconversationitemtruncated) | Server event when the client truncates an earlier assistant audio message item. |
| [RealtimeServerEventError](#realtimeservereventerror) | Server event when an error occurs. |
| [RealtimeServerEventInputAudioBufferCleared](#realtimeservereventinputaudiobuffercleared) | Server event when the client clears the input audio buffer. |
| [RealtimeServerEventInputAudioBufferCommitted](#realtimeservereventinputaudiobuffercommitted) | Server event when an input audio buffer is committed, either by the client or automatically in server VAD mode. |
| [RealtimeServerEventInputAudioBufferSpeechStarted](#realtimeservereventinputaudiobufferspeechstarted) | Server event in server turn detection mode when speech is detected. |
| [RealtimeServerEventInputAudioBufferSpeechStopped](#realtimeservereventinputaudiobufferspeechstopped) | Server event in server turn detection mode when speech stops. |
| [RealtimeServerEventRateLimitsUpdated](#realtimeservereventratelimitsupdated) | Emitted after every "response.done" event to indicate the updated rate limits. |
| [RealtimeServerEventResponseAudioDelta](#realtimeservereventresponseaudiodelta) | Server event when the model-generated audio is updated. |
| [RealtimeServerEventResponseAudioDone](#realtimeservereventresponseaudiodone) | Server event when the model-generated audio is done. Also emitted when a response is interrupted, incomplete, or canceled. |
| [RealtimeServerEventResponseAudioTranscriptDelta](#realtimeservereventresponseaudiotranscriptdelta) | Server event when the model-generated transcription of audio output is updated. |
| [RealtimeServerEventResponseAudioTranscriptDone](#realtimeservereventresponseaudiotranscriptdone) | Server event when the model-generated transcription of audio output is done streaming. Also emitted when a response is interrupted, incomplete, or canceled. |
| [RealtimeServerEventResponseContentPartAdded](#realtimeservereventresponsecontentpartadded) | Server event when a new content part is added to an assistant message item during response generation. |
| [RealtimeServerEventResponseContentPartDone](#realtimeservereventresponsecontentpartdone) | Server event when a content part is done streaming in an assistant message item. Also emitted when a response is interrupted, incomplete, or canceled. |
| [RealtimeServerEventResponseCreated](#realtimeservereventresponsecreated) | Server event when a new Response is created. The first event of response creation, where the response is in an initial state of "in_progress". |
| [RealtimeServerEventResponseDone](#realtimeservereventresponsedone) | Server event when a response is done streaming. Always emitted, no matter the final state. |
| [RealtimeServerEventResponseFunctionCallArgumentsDelta](#realtimeservereventresponsefunctioncallargumentsdelta) | Server event when the model-generated function call arguments are updated. |
| [RealtimeServerEventResponseFunctionCallArgumentsDone](#realtimeservereventresponsefunctioncallargumentsdone) | Server event when the model-generated function call arguments are done streaming. Also emitted when a response is interrupted, incomplete, or canceled. |
| [RealtimeServerEventResponseOutputItemAdded](#realtimeservereventresponseoutputitemadded) | Server event when a new output item is added to a response. |
| [RealtimeServerEventResponseOutputItemDone](#realtimeservereventresponseoutputitemdone) | Server event when an output item is done streaming. Also emitted when a response is interrupted, incomplete, or canceled. |
| [RealtimeServerEventResponseTextDelta](#realtimeservereventresponsetextdelta) | Server event when the model-generated text is updated. |
| [RealtimeServerEventResponseTextDone](#realtimeservereventresponsetextdone) | Server event when the model-generated text is done. Also emitted when a response is interrupted, incomplete, or canceled. |
| [RealtimeServerEventSessionCreated](#realtimeservereventsessioncreated) | Server event when a session is created. |
| [RealtimeServerEventSessionUpdated](#realtimeservereventsessionupdated) | Server event when a session is updated. |

### RealtimeServerEventConversationCreated

The server `conversation.created` event is returned right after session creation. One conversation is created per session.

#### Event structure

```json
{
  "type": "conversation.created",
  "conversation": {
    "id": "<id>",
    "object": "<object>"
  }
}
```

#### Properties

| Field | Type | Description | 
|-------|------|-------------| 
| type | string | The event type must be `conversation.created`. | 
| conversation | object | The conversation resource. | 

#### Conversation properties

| Field | Type | Description | 
|-------|------|-------------| 
| id | string | The unique ID of the conversation. | 
| object | string | The object type must be `realtime.conversation`. | 

### RealtimeServerEventConversationItemCreated

The server `conversation.item.created` event is returned when a conversation item is created. There are several scenarios that produce this event:
  - The server is generating a response, which if successful produces either one or two items, which is of type `message` (role `assistant`) or type `function_call`.
  - The input audio buffer is committed, either by the client or the server (in `server_vad` mode). The server takes the content of the input audio buffer and adds it to a new user message item.
  - The client sent a `conversation.item.create` event to add a new item to the conversation.

#### Event structure

```json
{
  "type": "conversation.item.created",
  "previous_item_id": "<previous_item_id>"
}
```

#### Properties

| Field | Type | Description | 
|-------|------|-------------| 
| type | string | The event type must be `conversation.item.created`. | 
| previous_item_id | string | The ID of the preceding item in the conversation context, allows the client to understand the order of the conversation. | 
| item | [RealtimeConversationResponseItem](#realtimeconversationresponseitem) | The item that was created. |

### RealtimeServerEventConversationItemDeleted

The server `conversation.item.deleted` event is returned when the client deleted an item in the conversation with a `conversation.item.delete` event. This event is used to synchronize the server's understanding of the conversation history with the client's view.

#### Event structure

```json
{
  "type": "conversation.item.deleted",
  "item_id": "<item_id>"
}
```

#### Properties

| Field | Type | Description | 
|-------|------|-------------| 
| type | string | The event type must be `conversation.item.deleted`. | 
| item_id | string | The ID of the item that was deleted. | 

### RealtimeServerEventConversationItemInputAudioTranscriptionCompleted

The server `conversation.item.input_audio_transcription.completed` event is the result of audio transcription for speech written to the audio buffer. 

Transcription begins when the input audio buffer is committed by the client or server (in `server_vad` mode). Transcription runs asynchronously with response creation, so this event can come before or after the response events.

Realtime API models accept audio natively, and thus input transcription is a separate process run on a separate speech recognition model, currently always `whisper-1`. Thus the transcript can diverge somewhat from the model's interpretation, and should be treated as a rough guide.

#### Event structure

```json
{
  "type": "conversation.item.input_audio_transcription.completed",
  "item_id": "<item_id>",
  "content_index": 0,
  "transcript": "<transcript>"
}
```

#### Properties

| Field | Type | Description | 
|-------|------|-------------| 
| type | string | The event type must be `conversation.item.input_audio_transcription.completed`. | 
| item_id | string | The ID of the user message item containing the audio. | 
| content_index | integer | The index of the content part containing the audio. | 
| transcript | string | The transcribed text. | 

### RealtimeServerEventConversationItemInputAudioTranscriptionFailed

The server `conversation.item.input_audio_transcription.failed` event is returned when input audio transcription is configured, and a transcription request for a user message failed. This event is separate from other `error` events so that the client can identify the related item.

#### Event structure

```json
{
  "type": "conversation.item.input_audio_transcription.failed",
  "item_id": "<item_id>",
  "content_index": 0,
  "error": {
    "code": "<code>",
    "message": "<message>",
    "param": "<param>"
  }
}
```

#### Properties

| Field | Type | Description | 
|-------|------|-------------| 
| type | string | The event type must be `conversation.item.input_audio_transcription.failed`. | 
| item_id | string | The ID of the user message item. | 
| content_index | integer | The index of the content part containing the audio. | 
| error | object | Details of the transcription error.<br><br>See nested properties in the next table.|

#### Error properties

| Field | Type | Description | 
|-------|------|-------------| 
| type | string | The type of error. | 
| code | string | Error code, if any. | 
| message | string | A human-readable error message. | 
| param | string | Parameter related to the error, if any. | 

### RealtimeServerEventConversationItemTruncated

The server `conversation.item.truncated` event is returned when the client truncates an earlier assistant audio message item with a `conversation.item.truncate` event. This event is used to synchronize the server's understanding of the audio with the client's playback.

This event truncates the audio and removes the server-side text transcript to ensure there's no text in the context that the user doesn't know about.

#### Event structure

```json
{
  "type": "conversation.item.truncated",
  "item_id": "<item_id>",
  "content_index": 0,
  "audio_end_ms": 0
}
```

#### Properties

| Field | Type | Description | 
|-------|------|-------------| 
| type | string | The event type must be `conversation.item.truncated`. | 
| item_id | string | The ID of the assistant message item that was truncated. | 
| content_index | integer | The index of the content part that was truncated. | 
| audio_end_ms | integer | The duration up to which the audio was truncated, in milliseconds. | 

### RealtimeServerEventError

The server `error` event is returned when an error occurs, which could be a client problem or a server problem. Most errors are recoverable and the session stays open.

#### Event structure

```json
{
  "type": "error",
  "error": {
    "code": "<code>",
    "message": "<message>",
    "param": "<param>",
    "event_id": "<event_id>"
  }
}
```

#### Properties

| Field | Type | Description | 
|-------|------|-------------| 
| type | string | The event type must be `error`. | 
| error | object | Details of the error.<br><br>See nested properties in the next table.| 

#### Error properties

| Field | Type | Description | 
|-------|------|-------------| 
| type | string | The type of error. For example, "invalid_request_error" and "server_error" are error types. |
| code | string | Error code, if any. | 
| message | string | A human-readable error message. | 
| param | string | Parameter related to the error, if any. | 
| event_id | string | The ID of the client event that caused the error, if applicable. | 

### RealtimeServerEventInputAudioBufferCleared

The server `input_audio_buffer.cleared` event is returned when the client clears the input audio buffer with a `input_audio_buffer.clear` event.

#### Event structure

```json
{
  "type": "input_audio_buffer.cleared"
}
```

#### Properties

| Field | Type | Description | 
|-------|------|-------------| 
| type | string | The event type must be `input_audio_buffer.cleared`. | 

### RealtimeServerEventInputAudioBufferCommitted

The server `input_audio_buffer.committed` event is returned when an input audio buffer is committed, either by the client or automatically in server VAD mode. The `item_id` property is the ID of the user message item created. Thus a `conversation.item.created` event is also sent to the client.

#### Event structure

```json
{
  "type": "input_audio_buffer.committed",
  "previous_item_id": "<previous_item_id>",
  "item_id": "<item_id>"
}
```

#### Properties

| Field | Type | Description | 
|-------|------|-------------| 
| type | string | The event type must be `input_audio_buffer.committed`. | 
| previous_item_id | string | The ID of the preceding item after which the new item is inserted. | 
| item_id | string | The ID of the user message item created. | 

### RealtimeServerEventInputAudioBufferSpeechStarted

The server `input_audio_buffer.speech_started` event is returned in `server_vad` mode when speech is detected in the audio buffer. This event can happen any time audio is added to the buffer (unless speech is already detected). 

> [!NOTE]
> The client might want to use this event to interrupt audio playback or provide visual feedback to the user.

The client should expect to receive a `input_audio_buffer.speech_stopped` event when speech stops. The `item_id` property is the ID of the user message item created when speech stops. The `item_id` is also included in the `input_audio_buffer.speech_stopped` event unless the client manually commits the audio buffer during VAD activation.

#### Event structure

```json
{
  "type": "input_audio_buffer.speech_started",
  "audio_start_ms": 0,
  "item_id": "<item_id>"
}
```

#### Properties

| Field | Type | Description | 
|-------|------|-------------| 
| type | string | The event type must be `input_audio_buffer.speech_started`. | 
| audio_start_ms | integer | Milliseconds from the start of all audio written to the buffer during the session when speech was first detected. This property corresponds to the beginning of audio sent to the model, and thus includes the `prefix_padding_ms` configured in the session. | 
| item_id | string | The ID of the user message item created when speech stops. | 

### RealtimeServerEventInputAudioBufferSpeechStopped

The server `input_audio_buffer.speech_stopped` event is returned in `server_vad` mode when the server detects the end of speech in the audio buffer. 

The server also sends a `conversation.item.created` event with the user message item created from the audio buffer.

#### Event structure

```json
{
  "type": "input_audio_buffer.speech_stopped",
  "audio_end_ms": 0,
  "item_id": "<item_id>"
}
```

#### Properties

| Field | Type | Description | 
|-------|------|-------------| 
| type | string | The event type must be `input_audio_buffer.speech_stopped`. | 
| audio_end_ms | integer | Milliseconds since the session started when speech stopped. This property corresponds to the end of audio sent to the model, and thus includes the `min_silence_duration_ms` configured in the session. | 
| item_id | string | The ID of the user message item created. | 

### RealtimeServerEventRateLimitsUpdated

The server `rate_limits.updated` event is emitted at the beginning of a response to indicate the updated rate limits. 

When a response is created, some tokens are reserved for the output tokens. The rate limits shown here reflect that reservation, which is then adjusted accordingly once the response is completed.

#### Event structure

```json
{
  "type": "rate_limits.updated",
  "rate_limits": [
    {
      "name": "<name>",
      "limit": 0,
      "remaining": 0,
      "reset_seconds": 0
    }
  ]
}
```

#### Properties

| Field | Type | Description | 
|-------|------|-------------| 
| type | string | The event type must be `rate_limits.updated`. | 
| rate_limits | array of [RealtimeServerEventRateLimitsUpdatedRateLimitsItem](#realtimeservereventratelimitsupdatedratelimitsitem) | The list of rate limit information. | 

### RealtimeServerEventResponseAudioDelta

The server `response.audio.delta` event is returned when the model-generated audio is updated.

#### Event structure

```json
{
  "type": "response.audio.delta",
  "response_id": "<response_id>",
  "item_id": "<item_id>",
  "output_index": 0,
  "content_index": 0,
  "delta": "<delta>"
}
```

#### Properties

| Field | Type | Description | 
|-------|------|-------------|
| type | string | The event type must be `response.audio.delta`. | 
| response_id | string | The ID of the response. | 
| item_id | string | The ID of the item. | 
| output_index | integer | The index of the output item in the response. | 
| content_index | integer | The index of the content part in the item's content array. | 
| delta | string | Base64-encoded audio data delta. | 

### RealtimeServerEventResponseAudioDone

The server `response.audio.done` event is returned when the model-generated audio is done. 

This event is also returned when a response is interrupted, incomplete, or canceled.

#### Event structure

```json
{
  "type": "response.audio.done",
  "response_id": "<response_id>",
  "item_id": "<item_id>",
  "output_index": 0,
  "content_index": 0
}
```

#### Properties

| Field | Type | Description | 
|-------|------|-------------|
| type | string | The event type must be `response.audio.done`. | 
| response_id | string | The ID of the response. | 
| item_id | string | The ID of the item. | 
| output_index | integer | The index of the output item in the response. | 
| content_index | integer | The index of the content part in the item's content array. | 

### RealtimeServerEventResponseAudioTranscriptDelta

The server `response.audio_transcript.delta` event is returned when the model-generated transcription of audio output is updated.

#### Event structure

```json
{
  "type": "response.audio_transcript.delta",
  "response_id": "<response_id>",
  "item_id": "<item_id>",
  "output_index": 0,
  "content_index": 0,
  "delta": "<delta>"
}
```

#### Properties

| Field | Type | Description | 
|-------|------|-------------|
| type | string | The event type must be `response.audio_transcript.delta`. | 
| response_id | string | The ID of the response. | 
| item_id | string | The ID of the item. | 
| output_index | integer | The index of the output item in the response. | 
| content_index | integer | The index of the content part in the item's content array. | 
| delta | string | The transcript delta. | 

### RealtimeServerEventResponseAudioTranscriptDone

The server `response.audio_transcript.done` event is returned when the model-generated transcription of audio output is done streaming. 

This event is also returned when a response is interrupted, incomplete, or canceled.

#### Event structure

```json
{
  "type": "response.audio_transcript.done",
  "response_id": "<response_id>",
  "item_id": "<item_id>",
  "output_index": 0,
  "content_index": 0,
  "transcript": "<transcript>"
}
```

#### Properties

| Field | Type | Description | 
|-------|------|-------------|
| type | string | The event type must be `response.audio_transcript.done`. | 
| response_id | string | The ID of the response. | 
| item_id | string | The ID of the item. | 
| output_index | integer | The index of the output item in the response. | 
| content_index | integer | The index of the content part in the item's content array. | 
| transcript | string | The final transcript of the audio. | 

### RealtimeServerEventResponseContentPartAdded

The server `response.content_part.added` event is returned when a new content part is added to an assistant message item during response generation.

#### Event structure

```json
{
  "type": "response.content_part.added",
  "response_id": "<response_id>",
  "item_id": "<item_id>",
  "output_index": 0,
  "content_index": 0
}
```

#### Properties

| Field | Type | Description | 
|-------|------|-------------|
| type | string | The event type must be `response.content_part.added`. | 
| response_id | string | The ID of the response. | 
| item_id | string | The ID of the item to which the content part was added. | 
| output_index | integer | The index of the output item in the response. | 
| content_index | integer | The index of the content part in the item's content array. | 
| part | [RealtimeContentPart](#realtimecontentpart) | The content part that was added. | 

#### Part properties

| Field | Type | Description | 
|------|------|-------------| 
| type | [RealtimeContentPartType](#realtimecontentparttype) |  | 

### RealtimeServerEventResponseContentPartDone

The server `response.content_part.done` event is returned when a content part is done streaming in an assistant message item. 

This event is also returned when a response is interrupted, incomplete, or canceled.

#### Event structure

```json
{
  "type": "response.content_part.done",
  "response_id": "<response_id>",
  "item_id": "<item_id>",
  "output_index": 0,
  "content_index": 0
}
```

#### Properties

| Field | Type | Description | 
|-------|------|-------------|
| type | string | The event type must be `response.content_part.done`. | 
| response_id | string | The ID of the response. | 
| item_id | string | The ID of the item. | 
| output_index | integer | The index of the output item in the response. | 
| content_index | integer | The index of the content part in the item's content array. | 
| part | [RealtimeContentPart](#realtimecontentpart) | The content part that is done. |  

#### Part properties

| Field | Type | Description | 
|------|------|-------------| 
| type | [RealtimeContentPartType](#realtimecontentparttype) |  | 

### RealtimeServerEventResponseCreated

The server `response.created` event is returned when a new response is created. This is the first event of response creation, where the response is in an initial state of `in_progress`.

#### Event structure

```json
{
  "type": "response.created"
}
```

#### Properties

| Field | Type | Description | 
|-------|------|-------------|
| type | string | The event type must be `response.created`. | 
| response | [RealtimeResponse](#realtimeresponse) | The response object. |

### RealtimeServerEventResponseDone

The server `response.done` event is returned when a response is done streaming. This event is always emitted, no matter the final state. The response object included in the `response.done` event includes all output items in the response, but omits the raw audio data.

#### Event structure

```json
{
  "type": "response.done"
}
```

#### Properties

| Field | Type | Description | 
|-------|------|-------------|
| type | string | The event type must be `response.done`. | 
| response | [RealtimeResponse](#realtimeresponse) | The response object. |

### RealtimeServerEventResponseFunctionCallArgumentsDelta

The server `response.function_call_arguments.delta` event is returned when the model-generated function call arguments are updated.

#### Event structure

```json
{
  "type": "response.function_call_arguments.delta",
  "response_id": "<response_id>",
  "item_id": "<item_id>",
  "output_index": 0,
  "call_id": "<call_id>",
  "delta": "<delta>"
}
```

#### Properties

| Field | Type | Description | 
|-------|------|-------------|
| type | string | The event type must be `response.function_call_arguments.delta`. | 
| response_id | string | The ID of the response. | 
| item_id | string | The ID of the function call item. | 
| output_index | integer | The index of the output item in the response. | 
| call_id | string | The ID of the function call. | 
| delta | string | The arguments delta as a JSON string. | 

### RealtimeServerEventResponseFunctionCallArgumentsDone

The server `response.function_call_arguments.done` event is returned when the model-generated function call arguments are done streaming. 

This event is also returned when a response is interrupted, incomplete, or canceled.

#### Event structure

```json
{
  "type": "response.function_call_arguments.done",
  "response_id": "<response_id>",
  "item_id": "<item_id>",
  "output_index": 0,
  "call_id": "<call_id>",
  "arguments": "<arguments>"
}
```

#### Properties

| Field | Type | Description | 
|-------|------|-------------|
| type | string | The event type must be `response.function_call_arguments.done`. | 
| response_id | string | The ID of the response. | 
| item_id | string | The ID of the function call item. | 
| output_index | integer | The index of the output item in the response. | 
| call_id | string | The ID of the function call. | 
| arguments | string | The final arguments as a JSON string. | 

### RealtimeServerEventResponseOutputItemAdded

The server `response.output_item.added` event is returned when a new item is created during response generation.

#### Event structure

```json
{
  "type": "response.output_item.added",
  "response_id": "<response_id>",
  "output_index": 0
}
```

#### Properties

| Field | Type | Description | 
|-------|------|-------------|
| type | string | The event type must be `response.output_item.added`. | 
| response_id | string | The ID of the response to which the item belongs. | 
| output_index | integer | The index of the output item in the response. | 
| item | [RealtimeConversationResponseItem](#realtimeconversationresponseitem) | The item that was added. |

### RealtimeServerEventResponseOutputItemDone

The server `response.output_item.done` event is returned when an item is done streaming. 

This event is also returned when a response is interrupted, incomplete, or canceled.

#### Event structure

```json
{
  "type": "response.output_item.done",
  "response_id": "<response_id>",
  "output_index": 0
}
```

#### Properties

| Field | Type | Description | 
|-------|------|-------------|
| type | string | The event type must be `response.output_item.done`. | 
| response_id | string | The ID of the response to which the item belongs. | 
| output_index | integer | The index of the output item in the response. | 
| item | [RealtimeConversationResponseItem](#realtimeconversationresponseitem) | The item that is done streaming. | 

### RealtimeServerEventResponseTextDelta

The server `response.text.delta` event is returned when the model-generated text is updated. The text corresponds to the `text` content part of an assistant message item.

#### Event structure

```json
{
  "type": "response.text.delta",
  "response_id": "<response_id>",
  "item_id": "<item_id>",
  "output_index": 0,
  "content_index": 0,
  "delta": "<delta>"
}
```

#### Properties

| Field | Type | Description | 
|-------|------|-------------|
| type | string | The event type must be `response.text.delta`. | 
| response_id | string | The ID of the response. | 
| item_id | string | The ID of the item. | 
| output_index | integer | The index of the output item in the response. | 
| content_index | integer | The index of the content part in the item's content array. | 
| delta | string | The text delta. | 

### RealtimeServerEventResponseTextDone

The server `response.text.done` event is returned when the model-generated text is done streaming. The text corresponds to the `text` content part of an assistant message item. 

This event is also returned when a response is interrupted, incomplete, or canceled.

#### Event structure

```json
{
  "type": "response.text.done",
  "response_id": "<response_id>",
  "item_id": "<item_id>",
  "output_index": 0,
  "content_index": 0,
  "text": "<text>"
}
```

#### Properties

| Field | Type | Description | 
|-------|------|-------------|
| type | string | The event type must be `response.text.done`. | 
| response_id | string | The ID of the response. | 
| item_id | string | The ID of the item. | 
| output_index | integer | The index of the output item in the response. | 
| content_index | integer | The index of the content part in the item's content array. | 
| text | string | The final text content. | 

### RealtimeServerEventSessionCreated

The server `session.created` event is the first server event when you establish a new connection to the Realtime API. This event creates and returns a new session with the default session configuration.

#### Event structure

```json
{
  "type": "session.created"
}
```

#### Properties

| Field | Type | Description | 
|-------|------|-------------|
| type | string | The event type must be `session.created`. | 
| session | [RealtimeResponseSession](#realtimeresponsesession) | The session object. | 

### RealtimeServerEventSessionUpdated

The server `session.updated` event is returned when a session is updated by the client. If there's an error, the server sends an `error` event instead.

#### Event structure

```json
{
  "type": "session.updated"
}
```

#### Properties

| Field | Type | Description | 
|-------|------|-------------|
| type | string | The event type must be `session.updated`. | 
| session | [RealtimeResponseSession](#realtimeresponsesession) | The session object. |

## Components

### RealtimeAudioFormat

**Allowed Values:**

* `pcm16` 
* `g711_ulaw` 
* `g711_alaw` 

### RealtimeAudioInputTranscriptionModel

**Allowed Values:**

* `whisper-1` 

### RealtimeAudioInputTranscriptionSettings

| Field | Type | Description | 
|-------|------|-------------|
| model | [RealtimeAudioInputTranscriptionModel](#realtimeaudioinputtranscriptionmodel) | The default `whisper-1` model is currently the only model supported for audio input transcription. | 


### RealtimeClientEvent

| Field | Type | Description | 
|-------|------|-------------|
| type | [RealtimeClientEventType](#realtimeclienteventtype) | The type of the client event. |
| event_id | string | The unique ID of the event. |

### RealtimeClientEventType

**Allowed Values:**

* `session.update` 
* `input_audio_buffer.append` 
* `input_audio_buffer.commit` 
* `input_audio_buffer.clear` 
* `conversation.item.create` 
* `conversation.item.delete` 
* `conversation.item.truncate` 
* `response.create` 
* `response.cancel` 

### RealtimeContentPart

| Field | Type | Description | 
|-------|------|-------------|
| type | [RealtimeContentPartType](#realtimecontentparttype) | The type of the content part. |

### RealtimeContentPartType

**Allowed Values:**

* `input_text` 
* `input_audio` 
* `text` 
* `audio` 

### RealtimeConversationItemBase

The item to add to the conversation.

### RealtimeConversationRequestItem

You use the `RealtimeConversationRequestItem` object to create a new item in the conversation via the [conversation.item.create](#realtimeclienteventconversationitemcreate) event.

| Field | Type | Description | 
|-------|------|-------------|
| type | [RealtimeItemType](#realtimeitemtype) | The type of the item. |
| id | string | The unique ID of the item. The ID can be specified by the client to help manage server-side context. If the client doesn't provide an ID, the server generates one. |

### RealtimeConversationResponseItem

The `RealtimeConversationResponseItem` object represents an item in the conversation. It's used in some of the server events, such as:

- [conversation.item.created](#realtimeservereventconversationitemcreated)
- [response.output_item.added](#realtimeservereventresponseoutputitemadded)
- [response.output_item.done](#realtimeservereventresponseoutputitemdone)
- [`response.created`](#realtimeservereventresponsecreated) (via the `response` property type [`RealtimeResponse`](#realtimeresponse))
- [`response.done`](#realtimeservereventresponsedone) (via the `response` property type [`RealtimeResponse`](#realtimeresponse))

| Field | Type | Description | 
|-------|------|-------------|
| object | string | The identifier for the returned API object.<br><br>Allowed values: `realtime.item` |
| type | [RealtimeItemType](#realtimeitemtype) | The type of the item.<br><br>Allowed values: `message`, `function_call`, `function_call_output` | 
| id | string | The unique ID of the item. The ID can be specified by the client to help manage server-side context. If the client doesn't provide an ID, the server generates one.<br><br>This property is nullable. |

### RealtimeFunctionTool

The definition of a function tool as used by the realtime endpoint.

| Field | Type | Description | 
|-------|------|-------------|
| type | string | The type of the tool.<br><br>Allowed values: `function` |
| name | string | The name of the function. |
| description | string | The description of the function. |
| parameters | object | The parameters of the function. |

### RealtimeItemStatus

**Allowed Values:**

* `in_progress` 
* `completed` 
* `incomplete` 

### RealtimeItemType

**Allowed Values:**

* `message` 
* `function_call` 
* `function_call_output` 

### RealtimeMessageRole

**Allowed Values:**

* `system` 
* `user` 
* `assistant` 

### RealtimeRequestAssistantMessageItem

| Field | Type | Description | 
|-------|------|-------------|
| role | string | The role of the message.<br><br>Allowed values: `assistant` | 
| content | array of [RealtimeRequestTextContentPart](#realtimerequesttextcontentpart) | The content of the message. |

### RealtimeRequestAudioContentPart

| Field | Type | Description | 
|-------|------|-------------|
| type | string | The type of the content part.<br><br>Allowed values: `input_audio` | 
| transcript | string | The transcript of the audio. |

### RealtimeRequestFunctionCallItem

| Field | Type | Description | 
|-------|------|-------------|
| type | string | The type of the item.<br><br>Allowed values: `function_call` | 
| name | string | The name of the function call item. |
| call_id | string | The ID of the function call item. |
| arguments | string | The arguments of the function call item. |
| status | [RealtimeItemStatus](#realtimeitemstatus) | The status of the item. |

### RealtimeRequestFunctionCallOutputItem

| Field | Type | Description | 
|-------|------|-------------|
| type | string | The type of the item.<br><br>Allowed values: `function_call_output` | 
| call_id | string | The ID of the function call item. |
| output | string | The output of the function call item. |

### RealtimeRequestMessageItem

| Field | Type | Description | 
|-------|------|-------------|
| type | string | The type of the item.<br><br>Allowed values: `message` | 
| role | [RealtimeMessageRole](#realtimemessagerole) | The role of the message. |
| status | [RealtimeItemStatus](#realtimeitemstatus) | The status of the item. |

### RealtimeRequestMessageReferenceItem

| Field | Type | Description | 
|-------|------|-------------|
| type | string | The type of the item.<br><br>Allowed values: `message` | 
| id | string | The ID of the message item. |

### RealtimeRequestSession

| Field | Type | Description | 
|-------|------|-------------|
| modalities | array | The modalities that the session supports.<br><br>Allowed values: `text`, `audio`<br/><br/>For example, `"modalities": ["text", "audio"]` is the default setting that enables both text and audio modalities. To enable only text, set `"modalities": ["text"]`. You can't enable only audio. |
| instructions | string | The instructions (the system message) to guide the model's text and audio responses.<br><br>Here are some example instructions to help guide content and format of text and audio responses:<br>`"instructions": "be succinct"`<br>`"instructions": "act friendly"`<br>`"instructions": "here are examples of good responses"`<br><br>Here are some example instructions to help guide audio behavior:<br>`"instructions": "talk quickly"`<br>`"instructions": "inject emotion into your voice"`<br>`"instructions": "laugh frequently"`<br><br>While the model might not always follow these instructions, they provide guidance on the desired behavior. |
| voice | [RealtimeVoice](#realtimevoice) | The voice used for the model response for the session.<br><br>Once the voice is used in the session for the model's audio response, it can't be changed. | 
| input_audio_format | [RealtimeAudioFormat](#realtimeaudioformat) | The format for the input audio. | 
| output_audio_format | [RealtimeAudioFormat](#realtimeaudioformat) | The format for the output audio. | 
| input_audio_transcription | [RealtimeAudioInputTranscriptionSettings](#realtimeaudioinputtranscriptionsettings) | The settings for audio input transcription.<br><br>This property is nullable. |
| turn_detection | [RealtimeTurnDetection](#realtimeturndetection) | The turn detection settings for the session.<br><br>This property is nullable. |
| tools | array of [RealtimeTool](#realtimetool) | The tools available to the model for the session. |
| tool_choice | [RealtimeToolChoice](#realtimetoolchoice) | The tool choice for the session. |
| temperature | number | The sampling temperature for the model. The allowed temperature values are limited to [0.6, 1.2]. Defaults to 0.8. |
| max_response_output_tokens | integer or "inf" | The maximum number of output tokens per assistant response, inclusive of tool calls.<br><br>Specify an integer between 1 and 4096 to limit the output tokens. Otherwise, set the value to "inf" to allow the maximum number of tokens.<br><br>For example, to limit the output tokens to 1000, set `"max_response_output_tokens": 1000`. To allow the maximum number of tokens, set `"max_response_output_tokens": "inf"`.<br><br>Defaults to `"inf"`. |

### RealtimeRequestSystemMessageItem

| Field | Type | Description | 
|-------|------|-------------| 
| role | string | The role of the message.<br><br>Allowed values: `system` |
| content | array of [RealtimeRequestTextContentPart](#realtimerequesttextcontentpart) | The content of the message. | 

### RealtimeRequestTextContentPart

| Field | Type | Description | 
|-------|------|-------------| 
| type | string | The type of the content part.<br><br>Allowed values: `input_text` |
| text | string | The text content. |

### RealtimeRequestUserMessageItem

| Field | Type | Description | 
|-------|------|-------------| 
| role | string | The role of the message.<br><br>Allowed values: `user` |
| content | array of [RealtimeRequestTextContentPart](#realtimerequesttextcontentpart) or [RealtimeRequestAudioContentPart](#realtimerequestaudiocontentpart) | The content of the message. |

### RealtimeResponse

| Field | Type | Description | 
|-------|------|-------------| 
| object | string | The response object.<br><br>Allowed values: `realtime.response` |
| id | string | The unique ID of the response. |
| status | [RealtimeResponseStatus](#realtimeresponsestatus) | The status of the response.<br><br>The default status value is `in_progress`. |
| status_details | [RealtimeResponseStatusDetails](#realtimeresponsestatusdetails) | The details of the response status.<br><br>This property is nullable. |
| output | array of [RealtimeConversationResponseItem](#realtimeconversationresponseitem) | The output items of the response. |
| usage | object | Usage statistics for the response. Each Realtime API session maintains a conversation context and appends new items to the conversation. Output from previous turns (text and audio tokens) is input for later turns.<br><br>See nested properties next.|
| + total_tokens | integer | The total number of tokens in the Response including input and output text and audio tokens.<br><br>A property of the `usage` object. | 
| + input_tokens | integer | The number of input tokens used in the response, including text and audio tokens.<br><br>A property of the `usage` object. |
| + output_tokens | integer | The number of output tokens sent in the response, including text and audio tokens.<br><br>A property of the `usage` object. | 
| + input_token_details | object | Details about the input tokens used in the response.<br><br>A property of the `usage` object.<br>br><br>See nested properties next.|
| + cached_tokens | integer | The number of cached tokens used in the response.<br><br>A property of the `input_token_details` object. | 
| + text_tokens | integer | The number of text tokens used in the response.<br><br>A property of the `input_token_details` object. | 
| + audio_tokens | integer | The number of audio tokens used in the response.<br><br>A property of the `input_token_details` object. | 
| + output_token_details | object | Details about the output tokens used in the response.<br><br>A property of the `usage` object.<br><br>See nested properties next.|
| + text_tokens | integer | The number of text tokens used in the response.<br><br>A property of the `output_token_details` object. | 
| + audio_tokens | integer | The number of audio tokens used in the response.<br><br>A property of the `output_token_details` object. | 


### RealtimeResponseAudioContentPart

| Field | Type | Description | 
|-------|------|-------------| 
| type | string | The type of the content part.<br><br>Allowed values: `audio` |
| transcript | string | The transcript of the audio.<br><br>This property is nullable. |

### RealtimeResponseBase

The response resource.

### RealtimeResponseFunctionCallItem

| Field | Type | Description | 
|-------|------|-------------| 
| type | string | The type of the item.<br><br>Allowed values: `function_call` |
| name | string | The name of the function call item. |
| call_id | string | The ID of the function call item. |
| arguments | string | The arguments of the function call item. |
| status | [RealtimeItemStatus](#realtimeitemstatus) | The status of the item. |

### RealtimeResponseFunctionCallOutputItem

| Field | Type | Description | 
|-------|------|-------------| 
| type | string | The type of the item.<br><br>Allowed values: `function_call_output` |
| call_id | string | The ID of the function call item. |
| output | string | The output of the function call item. |

### RealtimeResponseMessageItem

| Field | Type | Description | 
|-------|------|-------------| 
| type | string | The type of the item.<br><br>Allowed values: `message` |
| role | [RealtimeMessageRole](#realtimemessagerole) | The role of the message. |
| content | array | The content of the message.<br><br>Array items: [RealtimeResponseTextContentPart](#realtimeresponsetextcontentpart) |
| status | [RealtimeItemStatus](#realtimeitemstatus) | The status of the item. |

### RealtimeResponseOptions

| Field | Type | Description |
|-------|------|-------------|
| modalities | array | The modalities that the session supports.<br><br>Allowed values: `text`, `audio`<br/><br/>For example, `"modalities": ["text", "audio"]` is the default setting that enables both text and audio modalities. To enable only text, set `"modalities": ["text"]`. You can't enable only audio. |
| instructions | string | The instructions (the system message) to guide the model's text and audio responses.<br><br>Here are some example instructions to help guide content and format of text and audio responses:<br>`"instructions": "be succinct"`<br>`"instructions": "act friendly"`<br>`"instructions": "here are examples of good responses"`<br><br>Here are some example instructions to help guide audio behavior:<br>`"instructions": "talk quickly"`<br>`"instructions": "inject emotion into your voice"`<br>`"instructions": "laugh frequently"`<br><br>While the model might not always follow these instructions, they provide guidance on the desired behavior. |
| voice | [RealtimeVoice](#realtimevoice) | The voice used for the model response for the session.<br><br>Once the voice is used in the session for the model's audio response, it can't be changed. | 
| output_audio_format | [RealtimeAudioFormat](#realtimeaudioformat) | The format for the output audio. | 
| tools | array of [RealtimeTool](#realtimetool) | The tools available to the model for the session. |
| tool_choice | [RealtimeToolChoice](#realtimetoolchoice) | The tool choice for the session. |
| temperature | number | The sampling temperature for the model. The allowed temperature values are limited to [0.6, 1.2]. Defaults to 0.8. |
| max__output_tokens | integer or "inf" | The maximum number of output tokens per assistant response, inclusive of tool calls.<br><br>Specify an integer between 1 and 4096 to limit the output tokens. Otherwise, set the value to "inf" to allow the maximum number of tokens.<br><br>For example, to limit the output tokens to 1000, set `"max_response_output_tokens": 1000`. To allow the maximum number of tokens, set `"max_response_output_tokens": "inf"`.<br><br>Defaults to `"inf"`. |

### RealtimeResponseSession

| Field | Type | Description | 
|-------|------|-------------|
| object | string | The session object.<br><br>Allowed values: `realtime.session` |
| id | string | The unique ID of the session. | 
| model | string | The model used for the session. | 
| modalities | array | The modalities that the session supports.<br><br>Allowed values: `text`, `audio`<br/><br/>For example, `"modalities": ["text", "audio"]` is the default setting that enables both text and audio modalities. To enable only text, set `"modalities": ["text"]`. You can't enable only audio. |
| instructions | string | The instructions (the system message) to guide the model's text and audio responses.<br><br>Here are some example instructions to help guide content and format of text and audio responses:<br>`"instructions": "be succinct"`<br>`"instructions": "act friendly"`<br>`"instructions": "here are examples of good responses"`<br><br>Here are some example instructions to help guide audio behavior:<br>`"instructions": "talk quickly"`<br>`"instructions": "inject emotion into your voice"`<br>`"instructions": "laugh frequently"`<br><br>While the model might not always follow these instructions, they provide guidance on the desired behavior. |
| voice | [RealtimeVoice](#realtimevoice) | The voice used for the model response for the session.<br><br>Once the voice is used in the session for the model's audio response, it can't be changed. | 
| input_audio_format | [RealtimeAudioFormat](#realtimeaudioformat) | The format for the input audio. | 
| output_audio_format | [RealtimeAudioFormat](#realtimeaudioformat) | The format for the output audio. | 
| input_audio_transcription | [RealtimeAudioInputTranscriptionSettings](#realtimeaudioinputtranscriptionsettings) | The settings for audio input transcription.<br><br>This property is nullable. |
| turn_detection | [RealtimeTurnDetection](#realtimeturndetection) | The turn detection settings for the session.<br><br>This property is nullable. |
| tools | array of [RealtimeTool](#realtimetool) | The tools available to the model for the session. |
| tool_choice | [RealtimeToolChoice](#realtimetoolchoice) | The tool choice for the session. |
| temperature | number | The sampling temperature for the model. The allowed temperature values are limited to [0.6, 1.2]. Defaults to 0.8. |
| max_response_output_tokens | integer or "inf" | The maximum number of output tokens per assistant response, inclusive of tool calls.<br><br>Specify an integer between 1 and 4096 to limit the output tokens. Otherwise, set the value to "inf" to allow the maximum number of tokens.<br><br>For example, to limit the output tokens to 1000, set `"max_response_output_tokens": 1000`. To allow the maximum number of tokens, set `"max_response_output_tokens": "inf"`. |

### RealtimeResponseStatus

**Allowed Values:**

* `in_progress` 
* `completed` 
* `cancelled` 
* `incomplete` 
* `failed` 

### RealtimeResponseStatusDetails

| Field | Type | Description | 
|-------|------|-------------|
| type | [RealtimeResponseStatus](#realtimeresponsestatus) | The status of the response. |

### RealtimeResponseTextContentPart

| Field | Type | Description | 
|-------|------|-------------|
| type | string | The type of the content part.<br><br>Allowed values: `text` |
| text | string | The text content. |

### RealtimeServerEvent

| Field | Type | Description | 
|-------|------|-------------|
| type | [RealtimeServerEventType](#realtimeservereventtype) | The type of the server event. |
| event_id | string | The unique ID of the event. |

### RealtimeServerEventRateLimitsUpdatedRateLimitsItem

| Field | Type | Description | 
|-------|------|-------------|
| name | string | The rate limit property name that this item includes information about. | 
| limit | integer | The maximum configured limit for this rate limit property. | 
| remaining | integer | The remaining quota available against the configured limit for this rate limit property. | 
| reset_seconds | number | The remaining time, in seconds, until this rate limit property is reset. | 

### RealtimeServerEventType

**Allowed Values:**

* `session.created` 
* `session.updated` 
* `conversation.created` 
* `conversation.item.created` 
* `conversation.item.deleted` 
* `conversation.item.truncated` 
* `response.created` 
* `response.done` 
* `rate_limits.updated` 
* `response.output_item.added` 
* `response.output_item.done` 
* `response.content_part.added` 
* `response.content_part.done` 
* `response.audio.delta` 
* `response.audio.done` 
* `response.audio_transcript.delta` 
* `response.audio_transcript.done` 
* `response.text.delta` 
* `response.text.done` 
* `response.function_call_arguments.delta` 
* `response.function_call_arguments.done` 
* `input_audio_buffer.speech_started` 
* `input_audio_buffer.speech_stopped` 
* `conversation.item.input_audio_transcription.completed` 
* `conversation.item.input_audio_transcription.failed` 
* `input_audio_buffer.committed` 
* `input_audio_buffer.cleared` 
* `error` 

### RealtimeServerVadTurnDetection

| Field | Type | Description | 
|-------|------|-------------|
| type | string | The type of turn detection.<br><br>Allowed values: `server_vad` |
| threshold | number | The activation threshold for the server VAD turn detection. In noisy environments, you might need to increase the threshold to avoid false positives. In quiet environments, you might need to decrease the threshold to avoid false negatives.<br><br>Defaults to `0.5`. You can set the threshold to a value between `0.0` and `1.0`. |
| prefix_padding_ms | string | The duration of speech audio (in milliseconds) to include before the start of detected speech.<br><br>Defaults to `300`. |
| silence_duration_ms | string | The duration of silence (in milliseconds) to detect the end of speech. You want to detect the end of speech as soon as possible, but not too soon to avoid cutting off the last part of the speech.<br><br>The model will respond more quickly if you set this value to a lower number, but it might cut off the last part of the speech. If you set this value to a higher number, the model will wait longer to detect the end of speech, but it might take longer to respond. |

### RealtimeSessionBase

Realtime session object configuration.

### RealtimeTool

The base representation of a realtime tool definition.

| Field | Type | Description | 
|-------|------|-------------|
| type | [RealtimeToolType](#realtimetooltype) | The type of the tool. |

### RealtimeToolChoice

The combined set of available representations for a realtime tool_choice parameter, encompassing both string literal options like 'auto' and structured references to defined tools.

### RealtimeToolChoiceFunctionObject

The representation of a realtime tool_choice selecting a named function tool.

| Field | Type | Description | 
|-------|------|-------------|
| type | string | The type of the tool_choice.<br><br>Allowed values: `function` |
| function | object | The function tool to select.<br><br>See nested properties next.|
| + name | string | The name of the function tool.<br><br>A property of the `function` object. | 

### RealtimeToolChoiceLiteral

The available set of mode-level, string literal tool_choice options for the realtime endpoint.

**Allowed Values:**

* `auto` 
* `none` 
* `required` 

### RealtimeToolChoiceObject

A base representation for a realtime tool_choice selecting a named tool.

| Field | Type | Description | 
|-------|------|-------------|
| type | [RealtimeToolType](#realtimetooltype) | The type of the tool_choice. |

### RealtimeToolType

The supported tool type discriminators for realtime tools.
Currently, only 'function' tools are supported.

**Allowed Values:**

* `function` 

### RealtimeTurnDetection

| Field | Type | Description | 
|-------|------|-------------|
| type | [RealtimeTurnDetectionType](#realtimeturndetectiontype) | The type of turn detection.<br><br>Allowed values: `server_vad` |

### RealtimeTurnDetectionType

**Allowed Values:**

* `server_vad` 

### RealtimeVoice

**Allowed Values:**

* `alloy` 
* `shimmer` 
* `echo` 

## Related content

* Get started with the [Realtime API quickstart](./realtime-audio-quickstart.md).
* Learn more about [How to use the Realtime API](./how-to/realtime-audio.md).