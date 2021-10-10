# CS6349 - Project

This is a simple file storage project following the client-server model of communication written over plain TCP sockets with security enhancements satisfying the project requirements

### Tech

- Node.js - Socket programming
- RSA, SHA-256 and HMAC-SHA256
- Custom Encryption Algorithm using SHA-256 Algorithm

### How to run?

```sh
chmod +x ./cli.js
./cli.js server # Server
./cli.js upload ./file # Client Upload
./cli.js download ./file # Client Download
```

### Requirements and Fulfillments

1. File upload/download functionality
   - Implemented using the `TCP protocol` which provides connection packet order guarantees when compared to the UDP protocol
2. Authentication: The clients should be able to authenticate the server
   - We used `X.509 certificate` based authentication for the server
3. Confidentiality: Communication between the parties involved should be secure
   - Implemented using the `SHA-256` hashing algorithm
4. Integrity: Every packet should be verified for integrity
   - Implemented using the `HMAC-SHA256` algorithm
   - Protocol utilizes `round` and `counter` variables (separate for ingress and egress traffic) which are checked for each packet

### Protocol

##### Message Format

Before encryption, the packet stores the following information

1. `type`: Message Type
2. `data`: Message Data
3. `round`: An integer value which is incremented once every time the counter value needs to be reset
4. `counter`: An integer value used to keep track of the `nth` message sent/received on a connection

The digest is calculated by concatenating the original message fields (above) and sending it through the HMAC-SHA256 algorithm.
The payload along with the digest is encrypted using the custom encryption algorithm and sent on the connection.

##### Authentication

1. On initial connection from the client, the server sends out it's `X509 certificate` along with a `timestamp` signed with their private key
2. The client verifies the certificate with the CA and the timestamp with the public key of the server and `generates four session keys` (two for encryption and two for generating keyed-hashes) and responds (encrypted with the server's public key) with the following data
   - `32 byte random nonce` used as a `seed` to generate session keys
   - The `timestamp` sent by the server (Used by the server to validate the initial)
   - Additional details about the request such as `file metadata` etc.

##### Integrity

1. For integrity, we're using the Keyed Hash `HMAC-SHA-256` algorithm
2. The generated digest is added to the payload before encryption
3. After successful decryption, the processor verifies the data with the digest and breaks the connection if it fails

##### Confidentiality

1. For encryption, we use a custom encryption algorithm (a form of `stream cipher`)
2. Every packet is encrypted with a `Unique Key` which is generated by using `two values`
   1. `Key` takes from the generated keys or previously encrypted cipher text if it exists
   2. `IV` taken from the Signing Key else the current cipher key is used as the next round's IV
3. Decryption follows a similar pattern except in reverse

### Attack Scenarios and Mitigations

1. Passive Attacks
    1. Eavesdropping
        The communication protocol used is resistant against this kind of attack by using 4 keys, 2 for maintaining confidentiality and the other 2 for maintaining and the checking the integrity of the messages exchanged over the connection. These keys are used with SHA-256 algorithm for both encryption and message digest functions.
2. Active Attacks
    1. Replay Attacks
        The protocol used is resistant against these attacks by implementing a counter (unique to ingress and egress packets/messages) which is incremented after sending/receiving a message. The recipient compares the expected value with the received value to determine if it's replayed. Also, all the parties involved in the communication maintain a state which determines what type of messages it should expect. Any anomaly is treated as an attack and the connection is dropped.
    2. Message Manipulation Attacks
        The protocol is resistant against these types of attacks by implementing a message packing scheme where in, the sender generates the message digest by using the data (to be sent) + message type + round + counter values. The generated digest is appended to the message and the result is encrypted. The receiver will first decrypt the message and then check the round and counter values which are expected and then verify the integrity of the message received in a similar way.
    3. Node Compromise Attacks
        The protocol is also resistant to this kind of attack as the keys generated are ephemeral and are valid for only the timeline of the connection and not stored anywhere on the disk for an attacker to compromise the communication channel.
        If the server is compromised, the only way to recover is to revoke the certificate issued by the CA.
