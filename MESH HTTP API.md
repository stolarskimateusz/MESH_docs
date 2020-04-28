# MESH HTTP API

## Introduction
The Message Exchange for Social care and Health (MESH) component of the Spine allows messages and files to be delivered to registered recipients via a java client. Users register for a mailbox and install the client. The MESH client manages the sending of messages which users (typically user systems) have placed in an outbox directory on their machine.

It similarly manages the downloading of messages and files which other users have placed in the user’s virtual inbox on the Spine. The MESH server has been designed with an underlying HTTP based protocol. This protocol is capable of being used directly where user systems wish to bypass the MESH client or where they want to construct their own clients. Thus the MESH service will become a direct ‘store and retrieve’ messaging mechanism between endpoints as a new asynchronous pattern for Spine consumers. MESH uses a RESTful paradigm and thus messages which are send via mesh are POSTed to a MESH virtual outbox and recipients retrieve messages through a GET to their virtual inbox. The following document summarises the MESH HTTP API which may be used by clients directly or over which other SOAP or other APIs may be overlaid.

## Starting with MESH
1. Registering as an MESH endpoint
    <<TBD>>
2. Installing certificates
    <<TBD>>
3. Choosing how many mailboxes to create
    <<TBD>>

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

## HTTP messages
### Check user authentication
The Check User Authentication message attempts to connect to a specific mailbox. This allows the user to ensure that their authentication is correct and will update the details of the connection history held for the mailbox. It can be considered similar to a keep-alive or a ping message in that it allows monitoring on the Spine to be aware of the ongoing utilisation of the inbox despite a lack of traffic.

This message should be the first message sent in a ‘polling’ period for a mailbox. This allows the client to check that the MESH Server can be contacted and the mailbox is active before attempting to send/receive messages.

property|value
--- | ---
**URL**|/messageexchange/{mailboxId}
**HTTP Action**|POST
**Request Headers**|Authorization: [Authentication Headers (see below)]
|Mex-ClientVersion: {Client Version Number}
|Mex-OSArchitecture: {Operating System Architecture}
|Mex-OSName: {Operating System Name}
|Mex-OSVersion: {Operating System Version}
|Mex-JavaVersion: {JVM Version Number}
**Response Code**|200: Ok
|403: Authentication Failed
**Results**|If the authentication is successful then the Connection History on the Mailbox will be updated.

### Send message
This command uploads a message to the Message Exchange Server for onward delivery. The message is POST'ed to the Sender's Virtual Outbox on the Spine. Meta-information is supplied in the request headers and the content of the message is supplied in the request body. Messages will be kept for five days before being deleted at which point a ‘message undelivered message’ is sent to the sender for information.

If a message is more than 100Mb in size then it should be split into multiple chunks which are then compressed and uploaded separately. The first chunk in a large message are uploaded via this Send Message request which creates the message and returns the Message ID. This Message ID can then be use to upload the subsequent chunks. The message will only appear in the Recipient’s Inbox when the final chunk has been
successfully uploaded.

If senders have not received an acknowledgement within three days they MAY assume a failed delivery and consider re-sending. However the Spine does not determine fixed contract properties for MESH exchanges and users are free to agree alternative, more stringent, reliable exchange algorithms to suit their scenarios.

property|value
--- | ---
**URL**|/messageexchange/{senders mailbox ID}/outbox
**HTTP Action**|POST
**Request Headers**|Authorization: [Authentication Headers (see below)]
|Content-Type: application/octet-stream
|Mex-From: {senders mailbox ID}
|Mex-To: {recipient mailbox ID}
|Mex-WorkflowID: {DTS Workflow ID}
|Mex-FileName: {Original File Name}
|Mex-LocalID: {Local unique identifier of the message}
**Optional HTTP Request Headers**|Mex-ProcessID: {DTS Process ID}
|Mex-Content-Compress: {Flag to indicate that the contents have been automaticallycompressed by the client using GZip compression}
|Mex-Subject : {Subject line to be used for SMTP messages}
|Mex-Chunk-Range: Used if this is the first chunk of a large document. i.e. ‘1:n’ where n is the total number of chunks in the document.
|Content-Encoding: gzip
**Request Body**|The binary contents of the message which is being uploaded.
**Response Code**|202: Accepted
|403: Authentication Failed
|417: Invalid Recipient
**Response Body**|JSON which includes the Message ID of the newly created message record. {"messageID": allocatedMessageID}
|In the Case of a Validation Failure the JSON will contain additional elements which will provide information about the cause of the error:
|{"messageID": allocatedMessageID,
|"errorEvent": statusEvent,
|"errorCode" : statusCode,
|"errorDescription" : statusDescription}
|The errorEvent, errorCode & errorDescription entries will contain values as per the DTS Status Values in the DTS Client Interface Specification document.
**Results**|A new record is created in the messageExchangeRecord bucket which contains the meta-information about the message.
|The contents of the message if any is broken into 2Mb chunks and held in the messageExchangeChunk bucket.
|The Trading Summary Information for the Sending Mailbox is updated on the Spine so that the counts of the number of messages and the total message size transferred are updated. The ID of the newly created message is added to the inbox of the intended recipient mailbox.

### Upload Chunk
This command uploads additional chunks for large messages after the first chunk has been uploaded via the Send Message request. No Meta-Data for the message needs to be supplied as it has been supplied when the first chunk was uploaded.

The chunks should be uploaded to the Message ID which has been returned in the response for the Send Message. The Chunk No should start with 2 (as the first chunk has been uploaded with the Send Message request) and then incremented until all chunks have been successfully uploaded.

property|value
--- | ---
**URL**|/messageexchange/{senders mailbox ID}/outbox/{messageID}/chunkNo
**HTTP Action**|POST
**Request Headers**|Authorization: [Authentication Headers (see below)]
|Content-Type: application/octet-stream
|Mex-Chunk-Range: n:m
|Content-Encoding: gzip
**Request Body**|The binary contents of the compressed chunk which is being uploaded.
**Response Code**|202: Accepted
|403: Authentication Failed
**Response Body**|JSON which includes the Message ID of the newly created message record. {"messageID" : messageId, "blockId" : chunkNo}
**Results**|A new record is created in the messageExchangeRecord bucket which contains the meta-information about the message.
|The contents of the message if any is broken into 2Mb chunks and held in the messageExchangeChunk bucket.
|The Trading Summary Information for the Sending Mailbox is updated on the Spine so that the counts of the number of messages and the total message size transferred are updated. The ID of the newly created message is added to the inbox of the intended recipient mailbox.

### Check inbox
Checks if there are any messages that have been sent to a specific recipient mailbox and are ready to be downloaded. Client systems MUST poll their assigned inbox a minimum or once a day and a maximum of once every five minutes for messages. Any messages that are identified SHOULD be downloaded immediately. However, clients may choose to throttle messages or may be required to distribute processing time across a number of registered MESH mailboxes.

Only a list of the first 500 messages are returned. Therefore recipients SHOULD process the first 500 messages in their inbox prior to attempting to view the next awaiting messages.

property|value
--- | ---
**URL**|/messageexchange/{recipient}/inbox
**HTTP Action**|GET
**Request Headers**|Authorization: [Authentication Headers (see below)]
**Request Body**|The binary contents of the compressed chunk which is being uploaded.
**Response Code**|200: Ok
|403: Authentication Failed
**Response Body**|JSON containing the list of Message IDs in the recipient's inbox. (List is limited to the 1st 500 messages)
**Results**|No changes are made to the database by this action

