# ShopAI API Documentation `v0.2`

## Overview

The ShopAI backend is the core of ShopAI. Currently we are running one backend instance for each customer. Each instance has it's own address to connect to. The API of each instance enables the following functionalities:
- Initializing new chat sessions
- Sending messages
- Retrieving the chat history
- Retrieving the server status
- (Custom functionalities)

The communication with the backend is done via HTTP and Socket.IO.



## HTTP

### Initializing a new chat session

#### `GET /init_session`

- **Description**: Initializes a new chat session
- **Parameters**:
  - `license_key`: The instance license key
  - `lang`: The language of the chat session
- **Example**:
    ```
    GET /init_session?license_key=123456&lang=en
    ```
- **Response**:
    ```json
    {
        "status": "ok",
        "chat_token": "d42e7dc9-..."
    }
    ```
- **Errors**:
    - **Invalid language**:
        ```json
        {
            "status": "error",
            "message": "Invalide language, supported languages: ['hu', 'en']"
        }
        ```
    - **Invalid license key**:
        ```json
        {
            "status": "error",
            "message": "Invalid license key"
        }
        ```

### Retrieving the current version

#### `GET /version`
- **Response**:
    ```json
    {
        "status": "ok",
        "version": "v0.2"
    }
    ```

## Websocket

### Establishing a connection
- **Required query parameters**:
  - `chat_token`: The chat token received from the `/init_session` endpoint
- When successfully connected, the server will emit a `status` event with the status `operational`
- When `chat_token` is invalid, the server will reject the connection

### 💬 Client Events

All client events: "send_message", "get_history"

#### Sending a user message
**Trigger the server to process a user message**
> **Important:** Note the [status update section](#status-update). Block any input from the user by default until the server is `operational`. Do manually set status to `processing` directly after sending a message.

- Event name: `send_message`
- Event data: `String` `max 512 characters by default`
- Example:
    ```
    This is a message.
    ```

#### Retrieving the chat history
**Trigger the server to send the chat history**
- Event name: `get_history`
- Event data: `Array`
- Example:
    ```json
    [
        {
            "type": "ai",
            "content": "Hello!"
        },
        {
            "type": "user",
            "content": "Hi!"
        }
    ]
    ```

### 🗨️ Server Events

All server events: "history", "status", "token", "error"

#### History update
**Contains the full chat history**
- Event name: `history`
- Event data: `Array`
- Example:
    ```json
    [
        {
            "type": "ai",
            "content": "Hello!"
        },
        {
            "type": "user",
            "content": "Hi!"
        }
    ]
    ```
- How to handle:
    - Keep track of the chat history client-side
    - Always overwrite the current local chat history with the received one
    - Message types: `ai` (AI message), `user` (User message)

#### Status update
**Current status of the server**
- Event name: `status`
- Event data: `String`
- Example:
    ```
    operational
    ```
- How to handle:
  - Keep track of the server status client-side
  - Statuses: `operational`, `processing`
  - The server will automatically inform the client about the status changes
  - The server may not process incoming messages directly if it is currently in `processing` status
  - We recommend using other client-side statuses, such as `connecting`, `connected`, `disconnecting`, `disconnected`, `processing`

#### New Token String
**Contains the last generated token**
> **What is a token?** In the context of large language models (LLMs) like GPT, “tokens” refer to the pieces of text that the model processes. These can be words, parts of words, or punctuation marks, depending on how the model’s tokenizer breaks down the input text. Tokens are the basic units that the model uses to understand and generate language.

- Event name: `token`
- Event data: `String`
- Example:
    ```
    exam
    ```
    *Future tokens may be: "ple", " of", " a", " token". Concatenated this example would be "example of a token".*
- How to handle:
    - Check if the last chat message in history is an message of `"type": "ai"`
      - If not: Add a message with the type `ai` and the content of the token to the local chat history
      - If so: Concatenate the token to the last message of the local chat history
> **Important:** When the system is done generating the response, it will send a message of type `history` with the full chat history which contains the new message. The history should always overwrite the local chat history. The system will also send a message of type `status` with the status `operational`.

#### Error
**Contains an error message**
- Event name: `error`
- Event data: `String`
- The connection always closes after an error


