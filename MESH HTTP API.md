# MESH HTTP API

- [MESH HTTP API](#mesh-http-api)
  - [Introduction](#introduction)
  - [Starting with MESH](#starting-with-mesh)
  - [MESH Authorization Tokens](#mesh-authorization-tokens)
  - [MESH Polling Cycle](#mesh-polling-cycle)
  - [MESH usage requirements](#mesh-usage-requirements)
    - [Summary of MESH header attributes](#summary-of-mesh-header-attributes)
  - [HTTP messages](#http-messages)
    - [Authenticate Mailbox (handshake)](#authenticate-mailbox-handshake)
      - [Example call](#example-call)
    - [Send a message](#send-a-message)
      - [Example call](#example-call-1)
    - [Upload a Chunk](#upload-a-chunk)
    - [Check inbox](#check-inbox)
      - [Example call](#example-call-2)
    - [Check inbox count](#check-inbox-count)
    - [Check Message Status](#check-message-status)
    - [Download message](#download-message)
      - [Example call](#example-call-3)
    - [Download message chunk](#download-message-chunk)
    - [Acknowledge message download](#acknowledge-message-download)
      - [Example call](#example-call-4)


## Introduction
The Message Exchange for Social care and Health (MESH) component of the Spine allows messages and files to be delivered to registered recipients via a java client. Users register for a mailbox and install the client. The MESH client manages the sending of messages which users (typically user systems) have placed in an outbox directory on their machine.

It similarly manages the downloading of messages and files which other users have placed in the user’s virtual inbox on the Spine. The MESH server has been designed with an underlying HTTP based protocol. This protocol is capable of being used directly where user systems wish to bypass the MESH client or where they want to create their own clients. Thus the MESH service will become a direct 'store and retrieve' messaging mechanism between endpoints as a new asynchronous pattern for Spine consumers.

The Message Exchange HTTP API provides a RESTful interface to MESH to allow messages to be transferred between sender & recipient mailboxes.
This HTTP API is used by all Mailboxes that send and/or receive messages through Message Exchange. This includes the following:
- MESH Client - Sends HTTP Requests directly to the MESH HTTP API
- DTS Client - Sends messages to the DTS Adapter which then delegates the action via the MESH HTTP API
- 3rd Party Clients - External suppliers can create their own client implementations which upload & download files via the MESH HTTP API. (Right now this is limited to connections over the N3 network, but the work to allow this access via the internet is in procces.)
- 'Internal' Applications - Messages can be sent from core Spine processes to external mailboxes by uploading messages via the MESH HTTP API

## Starting with MESH
1. Registering as an MESH endpoint
   - Test
     * Integation Test Environment: https://msg.int.spine2.ncrs.nhs.uk/
     * Development Test Enivornment: https://nww.dev.spine2.ncrs.nhs.uk/
     * Deployment Test Environment: https://nww.dep.spine2.ncrs.nhs.uk/
   - Live
     * Live environment: https://mesh-sync.national.ncrs.nhs.uk/
2. Installing certificates
    <<TBD>>
3. Choosing how many mailboxes to create
    <<TBD>>

## MESH Authorization Tokens
Each message in the MESH REST API requires a custom authentication token to be passed in the Authorization Request Header. A new token must be generated for each HTTP Request, this can be acheived by regenerating it with a new Nonce or incremented NonceCount. The format of the Authorization token is the concatenation of the following elements (with a ':' separator between each element except after the schema name):
- 'NHSMESH' - The name of the Custom Authentication Schema. The Authentication Header should prefix the generated authentication token with the name of this schema followed by a space character.
- Mailbox ID - ID of the Mailbox sending the HTTP Request, must be uppercase.
- Nonce - A GUID used as an encryption 'Nonce'
- NonceCount - The number of times that the same 'Nonce' has been used.
- Timestamp - The current date and time in 'yyyyMMddHHmm' format.
- Hash Code - HMAC-SHA256 Hash Code generated from the concatenation of the following elements, joined by ':'
  * Mailbox ID - As above
  * Nonce - As above
  * Nonce Count - As above
  * Mailbox Password - The password for the MESH Mailbox.
  * Timestamp - As above

The following Request Header MUST be present in every HTTP Request:
`Authorization: NHSMESH MAILBOX01:73eefd69-811f-44d0-81f8-a54ff352a991:001:201511041205:3097fd5aa85a...f540942614b`\
NB - The Shared Key to generate the HMAC-SHA256 hash will need to be manually requested from HSCIC.\
Websites are available to generate both GUIDs and HMAC-SHA256 hashes if required.

The Server will perform the following checks to authenticate the request:
- Get the User ID from the Recipient/Sender ID
- Get the expected password from the MESH database
- Use HMAC-SHA256 on the User ID, Password and the Nonce & Nonce Count from the Request Header and check that the hashed values match. The Secret Key for the HMAC generation will be supplied by HSCIC on request.
- Check that we have not already received this combination of User ID, Nonce & Nonce Count on a previous HTTP request.
- Check that we have not already received this combination of User ID, Nonce & Nonce Count on a previous HTTP request. (Use a method similar to EBXml Dedupe to hold these values in a Redis DB?)
- Check that the Timestamp is within allowable bounds (e.g. ± 2 hour of current time to allow for BST/GMT & client/server clock differences)

## MESH Polling Cycle
Each Mailbox should periodically check to see if any outbound files need to be sent via MESH and check to see if there are any files which have been sent to the Mailbox to download. If no messages are sent or received for a mailbox during a polling period then the mailbox should pause for a number of minutes (typically 5, 10, 15 minutes) before starting a subsequent polling cycle for the mailbox.

The basic pseudo-code for a mailbox’s polling cycle is as follows:
- Authenticate Mailbox
- For each file waiting to be sent:
  - Split the message into chunks of <100MB
  - Send file to recipient indicating number of chunks
    - Upload message chunk
- Fetch the list of messages in the Mailbox’s Inbox
  - For each available message:
    - Download the first part of the message
    - If identified as a chunked message
      - Save the first message chunk to the file system
      - For each of the remaining chunks
        - Download message chunk
        - Save the message chunk to the file system
  - Acknowledge the successful receipt of the message

## MESH usage requirements
MESH usage requirements:
   - MESH messages MUST be less than 100MB. Messages over 100MB will be rejected with an error. However, note that the DDC has started considering chunking approaches to increase maximum message sizes.
   - Although the MESH is a highly reliable service, message exchange clients are responsible for end to end reliability of exchanges.
   - Endpoints
The basic pseudocode for a mailbox’s polling cycle is as follows:
- Authenticate Mailbox
- For each file waiting to be sent:
  * Send file to recipient
- Fetch the list of messages in the Mailbox’s Inbox
- For each available message:
  * Download the message
  * Save the message to the file system.
  * Acknowledge the successful receipt of the message

The following sequence diagram shows a typical MESH message exchange.

**[Diagram here]**

### Summary of MESH header attributes

Attribute|Purpose|Mandatory|Legacy
--- | ---| ---| ---
Authorization:|To authenticate sender|Yes|
Accept-Encoding:|indicates if content is compressed, in our case the most common use is "gzip"|No|
Mex-ClientVersion:|For audit purposes|No|
Mex-OSArchitecture:|For audit purposes|No|
Mex-OSName:|For audit purposes|No|
Mex-OSVersion:|For audit purposes|No|
Mex-JavaVersion:|For audit purposes|No|
Mex-From:|Sender of the message, a DTS address|Yes|
Mex-To:|Recipient of the message, a DTS address|Yes|
Mex-Version:||No||
Mex-ToSMTP:||No|
Mex-FromSMTP:||No|
Mex-WorkflowID:|Identifies the type of message being sent e.g. Pathology, GP Capitation|No|
Mex-FileName:|The name of the attached file|No|
Mex-ProcessID:|For future use. Identifier to specify the type of processing that might be required before forwarding to the recipient.|No|
Mex-Content-Compress:|Flag to indicate that the contents have been automatically compressed by the client using GZip compression|No|Yes
Mex-Content-Encrypted:|Flag indicating that the original message is encrypted|No|Yes
Mex-Content-Compressed:|Flag indicating that the original message is encrypted|No|Yes
Mex-Content-Checksum:|Checksum of the original message contents|No|Yes
Mex-MessageType:|'Data' or 'Report'|No|Yes
Mex-LocalID:|Local unique identifier of the message|No|
Mex-Subject:|Subject line to be used for SMTP messages|No|
Mex-PartnerID:|Obsolete|No|Yes
Mex-MessageID:|Identifies a message once it has first been uploaded to the MESH server|Yes|
Mex-Chunk-Range|Used for Chunked Messages and indicates the Chunk No & the total number of chunks in the document. Formatted as n:m: n - Chunk No, m - Total No of Chunks|No

## HTTP messages
### Authenticate Mailbox (handshake)
It performs an authentication check on the Mailbox. This should be performed before the mailbox attempts to send or receive files. This action should be the first action in each polling cycle.

The Check User Authentication message attempts to connect to a specific mailbox. This allows the user to ensure that their authentication is correct and will update the details of the connection history held for the mailbox. It can be considered similar to a keep-alive or a ping message in that it allows monitoring on the Spine to be aware of the ongoing utilisation of the inbox despite a lack of traffic.

This message should be the first message sent in a ‘polling’ period for a mailbox. This allows the client to check that the MESH Server can be contacted and the mailbox is active before attempting to send/receive messages.

property|value
--- | ---
**URL**|/messageexchange/{mailboxId}
**HTTP Action**|POST
**Request Headers**|Authorization: [Authentication Headers (see below)]<br>Mex-ClientVersion: {Client Version Number}<br>Mex-OSArchitecture: {Operating System Architecture}<br>Mex-OSName: {Operating System Name}<br>Mex-OSVersion: {Operating System Version}<br>Mex-JavaVersion: {JVM Version Number}
**Response Code**|200: Ok<br>403: Authentication Failed
**Results**|If the authentication is successful then the Connection History on the Mailbox will be updated.

#### Example call
```
POST https://10.210.162.216/messageexchange/NONFUNC01

POST data:

Request Headers:
  Connection: keep-alive
  Mex-JavaVersion: 1.7.0_60
  Mex-ClientVersion: Alpha.0.0.1
  Mex-OSVersion: 6.1
  Mex-OSArchitecture: Windows 10
  Mex-OSName: x86_64
  Authorization: NONFUNC01:jt81ti68rlvta7379p3ng949rv:1:201511201038:291259e6a73cde3278f99cd5bd3c7ec9b3d1d5479077a8f711bddf58073d5555
  Content-Type: application/x-www-form-urlencoded
```

### Send a message
This command uploads a message to the Message Exchange Server for onward delivery. The message is POSTed to the Sender's Virtual Outbox on the Spine. Meta-information is supplied in the request headers and the content of the message is supplied in the request body. Messages will be kept for five days before being deleted at which point a ‘message undelivered message’ is sent to the sender for information.

If senders have not received an acknowledgement within three days they *may* assume a failed delivery and consider re-sending. However the Spine does not determine fixed contract properties for MESH exchanges and users are free to agree alternative, more stringent, reliable exchange algorithms to suit their scenarios.

If a message is more than 100Mb in size then it should be split into multiple chunks which are then compressed and uploaded separately. The first chunk in a large message are uploaded via this Send Message request, with the addition of the Mex-Chunk-Range header which indicates that it is chunk 1 of n, which creates the message and returns the Message ID. The subsequent chunks with the obtained Message ID are then uploaded using individual the Upload Chunk requests. The message will only appear in the Recipient’s Inbox when the final chunk has been
successfully uploaded. 

**Important** The support for large chunked messages is optional and so it is important to ensure that intended recipient of the large file can accept chunked files.

property|value
--- | ---
**URL**|/messageexchange/{senders mailbox ID}/outbox
**HTTP Action**|POST
**Request Headers**|Authorization: [Authentication Headers (see below)]<br>Content-Type: application/octet-stream<br>Mex-From: {senders mailbox ID}<br>Mex-To: {recipient mailbox ID}<br>Mex-WorkflowID: {Workflow ID}<br>Mex-FileName: {Original File Name}<br>Mex-LocalID: {Local unique identifier of the message}
**Optional HTTP Request Headers**|Mex-ProcessID: {Process ID}<br>Mex-Content-Compress: {Flag to indicate that the contents have been automaticallycompressed by the client using GZip compression}<br>Mex-Subject : {Subject line to be used for SMTP messages}<br>Mex-Chunk-Range: Used if this is the first chunk of a large document. i.e. ‘1:n’ where n is the total number of chunks in the document.<br>Content-Encoding: gzip
**Request Body**|The binary contents of the message which is being uploaded.
**Response Code**|202: Accepted<br>403: Authentication Failed<br>417: Invalid Recipient
**Response Body**|JSON which includes the Message ID of the newly created message record. {"messageID": allocatedMessageID}<br>In the Case of a Validation Failure the JSON will contain additional elements which will provide information about the cause of the error:<br>{"messageID": allocatedMessageID,<br>"errorEvent": statusEvent,<br>"errorCode" : statusCode,<br>"errorDescription" : statusDescription}<br>The errorEvent, errorCode & errorDescription entries will contain values as per the Status Values in the Client Interface Specification document.
**Results**|A new record is created in the messageExchangeRecord bucket which contains the meta-information about the message.<br>The contents of the message if any is broken into 2Mb chunks and held in the messageExchangeChunk bucket.<br>The Trading Summary Information for the Sending Mailbox is updated on the Spine so that the counts of the number of messages and the total message size transferred are updated. The ID of the newly created message is added to the inbox of the intended recipient mailbox.

#### Example call
```
POST https://10.210.162.216/messageexchange/NONFUNC01/outbox

POST data:
  This is a test file
  I am typing this so I have a file to send via MESH

Request Headers:
  Connection: keep-alive
  Authorization: NONFUNC01:jt81ti68rlvta7379p3ng949rv:8:201511201038:e1672428d9f02e6d14b686bb7953e065eaf7bb146817e9016cf5974612953f5f
  Content-Type: application/octet-stream
  Mex-From: NONFUNC01
  Mex-To: NONFUNC02
  Mex-WorkflowID: 2015_11_12_1543
  Mex-Content-Compress: N
  Mex-Content-Encrypted: N
  Mex-Content-Compressed: Y
  Mex-MessageType: Data
  Mex-LocalID: TESTING01
  Mex-Subject: TEST TRANSFER 01
```

### Upload a Chunk
This command uploads additional chunks for large messages after the first chunk has been uploaded via the Send Message request. No Meta-Data for the message needs to be supplied as it has been supplied when the first chunk was uploaded.

The chunks should be uploaded to the Message ID, which has been returned in the response for the Send Message. The Chunk No should start with 2 (as the first chunk has been uploaded with the Send Message request) and then incremented until all chunks have been successfully uploaded.

**IMPORTANT** The original large file should be broken into chunks and then each chunk will be individually compressed. The each chunk MUST use the same compression alogorithm as the initial chunk uploaded via the Send Message request (e.g. all chunks should use gzip compression).

property|value
--- | ---
**URL**|/messageexchange/{senders mailbox ID}/outbox/{messageID}/{chunkNo}
**HTTP Action**|POST
**Request Headers**|Authorization: [Authentication Headers (see below)]<br>Content-Type: application/octet-stream<br>Mex-Chunk-Range: n:m<br>Content-Encoding: gzip
**Request Body**|The binary contents of the compressed chunk which is being uploaded.
**Response Code**|202: Accepted<br>403: Authentication Failed
**Response Body**|JSON which includes the Message ID of the newly created message record. {"messageID" : messageId, "blockId" : chunkNo}
**Results**|A new record is created in the messageExchangeRecord bucket which contains the meta-information about the message.<br>The contents of the message if any is broken into 2Mb chunks and held in the messageExchangeChunk bucket.<br>The Trading Summary Information for the Sending Mailbox is updated on the Spine so that the counts of the number of messages and the total message size transferred are updated. The ID of the newly created message is added to the inbox of the intended recipient mailbox.

### Check inbox
Checks if there are any messages that have been sent to a specific recipient mailbox and are ready to be downloaded. Client systems **must** poll their assigned inbox minimum once a day and maximum once every five minutes for messages. Any messages that are identified **should** be downloaded immediately. However, clients may choose to throttle messages or may be required to distribute processing time across a number of registered MESH mailboxes.

Only a list of the first 500 messages are returned. Therefore recipients **should** process the first 500 messages in their inbox prior to attempting to view the next awaiting messages.

property|value
--- | ---
**URL**|/messageexchange/{recipient}/inbox
**HTTP Action**|GET
**Request Headers**|Authorization: [Authentication Headers (see below)]
**Response Code**|200: Ok<br>403: Authentication Failed
**Response Body**|JSON containing the list of Message IDs in the recipient's inbox. (List is limited to the 1st 500 messages)
**Results**|No changes are made to the database by this action

#### Example call
```
GET https://10.210.162.216/messageexchange/NONFUNC01/inbox

Request Headers:
  Connection: keep-alive
  Authorization: NONFUNC01:jt81ti68rlvta7379p3ng949rv:2:201511201038:2ba994afb852c7e0d49593f454ad1f8705f5729454958955fe188f191442ea79
```

### Check inbox count
This command returns the number of messages currently held in the MESH mailbox that are ready to download.

property|value
--- | ---
**URL**|/messageexchange/{recipient}/count
**HTTP Action**|GET
**Request Headers**|Authorization: [Authentication Headers (see below)]
**Response Code**|200: Ok<br>403: Authentication Failed
**Response Body**|JSON containing the list of Message IDs in the recipient's inbox. (List is limited to the 1st 500 messages) {"messageCount": MessageCount}
**Response Headers**|Content-Type: application/json

### Check Message Status
This command checks the status of a message from a specific recipient mailbox. This message replaces the DTS Message Tracking API call. The LocalID value is an optional parameter in MESH message.

property|value
--- | ---
**URL**|/messageexchange/{senders_mailbox_ID}/outbox/tracking/{localId}
**HTTP Action**|GET
**Request Headers**|Authorization: [Authentication Headers (see below)]
|Content-Type: application/octet-stream

### Download message
Retrieve a message based on its message_id.

property|value
--- | ---
**URL**|/messageexchange/{recipient}/inbox/{messageID}
**HTTP Action**|GET
**Request Headers**|Authorization: [Authentication Headers (see below)]<br>Accept-Encoding: (optional) 'gzip' if client can accept & decompress messages in GZip format
**Response Code**|200: Ok<br>206: Partial Download. Indicates that this is the 1 st chunk of a multi-chunk message. The subsequent chunks can be downloaded via calls to the Download Chunk request. (see section 4.6)<br>403: Authentication Failed<br>404: Message does not exist<br>410: Gone, message has already been downloaded
**Response Headers**|Content-Type: application/octet-stream<br>Content-Encoding: gzip if message was compressed on upload and can be accepted by the recipient.<br>Mex-From: {senders mailbox ID}<br>Mex-To: {recipient identifier}<br>Mex-WorkflowID: {DTS Workflow ID}<br>Mex-MessageID: {DTS Message ID}<br>Mex-FileName: {Original File Name}
**Optional HTTP Response Headers**|Mex-ProcessID: {DTS Workflow ID}<br>Mex-Content-Compress: {Flag to indicate that contents should be compressed during transport}<br>Mex-Chunk-Range: 1:n - Only returned for a large, multi-chunk message to indicate that this is the 1st chunk of n.
**Response Body**|The binary contents of the message which is being downloaded. This will be empty if the message is a Report 6 rather than Data.
**Results**|No changes are made to the database by this action.

#### Example call
```
GET https://10.210.162.216/messageexchange/NONFUNC01/inbox/20151120103837238795_386EAF_1429036893

Request Headers:
  Connection: keep-alive
  Authorization: NONFUNC01:jt81ti68rlvta7379p3ng949rv:6:201511201038:a764f7b9d0f9aab1ddfb4d9fe258ef63bc547094044408ae4a4dc0f036f06bae
```

### Download message chunk
Retrieve a chunck of the message based on its message_id and chunck number.
The lowest number is 2, as the first part has been retrieved since the download of the message.

property|value
--- | ---
**URL**|/messageexchange/{recipient}/inbox/{messageID}/{chunkNo}
**HTTP Action**|GET
**Request Headers**|Authorization: [Authentication Headers (see below)]
**Response Code**|200 : Ok – Indicates that the chunk has been downloaded successfully and that this was the final chunk in the message.<br>206: Partial Download – Indicates that chunk has been downloaded successfully and that there are further chunks.<br>403: Authentication Failure<br>404: Not Found – Indicates that the message chunk does not exist
**Response Headers**|Content-Type: application/octet-stream<br>Content-Encoding: gzip if message was compressed on upload and can be accepted by the recipient.<br>Mex-Chunk-Range: e.g. ‘2:4’ In the format n:m which indicates that this is chunk n of m chunks.<br>Mex-From: {senders mailbox ID}
**Response Body**|The contents of the downloaded chunk
**Results**|Update the details of the downloaded message to set the status and the download time.<br>Remove the Message ID from the Recipient's Inbox<br>Update the Trading Summary Information for the Recipient Mailbox to increment the download counter and the total number of bytes downloaded.

### Acknowledge message download
Acknowledge the successful download of a message. This will remove the message from the Recipient's Inbox.

This message **must** be sent by the recipient to indicate that a message has been downloaded and saved correctly. This changes the status of the message and removes it from the recipient's inbox. Note that this acknowledgement closes the transaction on the Spine,acknowledgement but does not result in an associated acknowledgement message to the sending system.

Senders will receive notification of undelivered messages after five days. Senders *may* choose to check on the delivery status of a message in the meantime using the tracing API.

property|value
--- | ---
**URL**|/messageexchange/{recipient}/inbox/{messageID}/status/acknowledged
**HTTP Action**|PUT
**Request Headers**|Authorization: [Authentication Headers (see below)]
**Response Code**|200 : Ok<br>403: Authentication Failure
**Results**|Update the details of the downloaded message to set the status and the download time.<br>Remove the Message ID from the Recipient's Inbox<br>Update the Trading Summary Information for the Recipient Mailbox to increment the download counter and the total number of bytes downloaded.

#### Example call
```
PUT https://10.210.162.216/messageexchange/NONFUNC01/inbox/20151120103837238795_386EAF_1429036893/status/acknowledged

PUT data:

Request Headers:
  Connection: keep-alive
  Authorization: NONFUNC01:jt81ti68rlvta7379p3ng949rv:7:201511201038:e0cc8fb33674c75da29f1cace36294974e3b748eda1b75561c5cd4bd89a37d09
```
