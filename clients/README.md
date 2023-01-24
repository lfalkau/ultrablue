# Ultrablue clients

Two clients are available from the ANSSI-FR repository: One for [IOS](ios/), and one for [Android](Android/). However, I'm not a mobile developer, and even if I tried to do my best on these apps, they could be way better.

This document describes how to implement a client for the Ultrablue [server](../server/). It allows anyone wanting to implement its own client to do so, without the need of going through the whole server code.

If you do, feel free to contribute on the project, pull requests are welcome!

---

# Messages overview:

The following diagram represents the message sequence exchanged between the server and the client. It reads from top to bottom.
Note that this diagram comes with datatypes annoted next to the message, in bold text. These types are described further in the **Data types** section.
Also note that some very important technical details are ommited from this diagram because they're not directly communication related. These details are addressed in the **Technical considerations** section.

![ultrablue message protocol](../doc/ressources/ultrablue_messages_protocol.svg)

> **Grey arrow**: Message displayed over a QR code
> **Blue arrow**: Message sent over Bluetooth Low Energy (BLE)
> 
> **Grey text**: JSON encoded
> **Black text**: CBOR encoded
> **Gold text**: CBOR + AES256/GCM encoded
> 
> **Green box**: Messages only sent at enroll time.

For a more general view of the protocol, you can see the [protocol](../doc/protocol/) section.

# Technical considerations:

Here are some notes on important things to know when implementing a client:

### The client UUID
It must be generated randomly at enroll time before being sent to the server. During later attestations, the same UUID needs to be sent by the client, to allow the server to fetch the shared key. If the server receives an UUID it already "knows" during an enrollment, it will fail and disconnect the client. Appropriate error handling must be done on client side (e.g. reconnect and generate a new UUID or tell the user why it failed and ask them to manually restart the enrollment).

### Encryption IVs
As described in the encoding section, most of messages are encrypted. When they are, the IV prefixes the encrypted message. IV must **NEVER** be reused with the same key. It must be randomly (and securely) generated each time.

### Tweaked auth nonce
During the authentication process, the server sends a nonce to the client. The client must tweak it and send it back. It needs to be tweaked because else, nothing would avoid a client to send back the exact same message it just received. It would be the same nonce, encrypted with the right key and a valid IV, and the server would authenticate it.
The chosen tweak is to split the nonce in two halfs, 8 bytes each, and to invert those. In python, this would give: `tweaked = nonce[8:] + nonce[:8]`.

### Credential activation
When sending the MakeCredential challenge to the attester's TPM, the nonce must be randomly generated, and when the challenge response comes back, the client must ensure it matches the generated one. This step proves to the verifier (client) that the Attestation Key you got has been generated on the same physical TPM than the one you enrolled and is not a replayed one (thanks to the nonce). Relying on external TPM libraries can be helpful to avoid doing things wrong: we used [go-attestation](https://pkg.go.dev/github.com/google/go-attestation) for our clients.

### Attestation process
When the servers sends the **PlatformParameters** object, several things needs to be done.
- In order to trust the attestation data just received, you need to verify both the quote signature (it is signed with the previously obtained Attestation Key), and the anti-replay nonce you previously sent.
- Once you know you can trust the quote, it's time to apply your **PCR policy**. It can be application or user defined. To do so, match the chosen PCR final digests of the signed quote against the reference ones, that you saved at enroll time. (This means that - at enroll time - you just have to save the PCRs without comparing them. This is a trust on first use model). If they match, nothing changed from the reference boot state: You can move on. If they differs however, you can decide what to do. The event log can help you with this decision, but it can't be trusted yet.
- In order to trust the event log, you must replay it. This consists in taking each individual events in it, and chain each in the same way the TPM would have done this at boot time. You must then compare the final digest from each PCR with the ones in the quote. You can only trust the event log and use it to make a decision if these digests matches. This needs to be done because the eventlog isn't signed, and can have been tampered with.

### Attestation response
Ultrablue can be used to do disk encryption. At enroll time, a flag is sent by the server to indicate weither you must send back a secret or not on attestation success. If you have to, this secret must be securely generated and securely stored. You'll need to send this secret back on later attestations for the same attester.

### Storing attester devices
To be able to perform remote attestation and act as a verifier, data about each attester must be stored at enroll time, including:
- Its TPM Endorsement Key (public part)
- BLE connection information (MAC address, advertised name...)
- The UUID you generated for this attester at enroll time
- The response attestation secret you generated at enroll time if you did
- The shared symmetric key used to encrypt the communication

Secrets should be securely stored, e.g. on the keystore for Android or on keychain for IOS (the secure enclave cannot be used to store those secrets, unfortunately).

Storing these information must be done at the right time: If you store them too early, you'll have to manually delete them if an error occurs. No device should be stored if an error occurs: The enrollment needs to be performed again.

# Data types

Here is the list of data structures used by the protocol. Some are ours, others are from the [go-attestation package](https://pkg.go.dev/github.com/google/go-attestation/attest). Structures are written in go here, but when implementing a client, you may need to adapt them, according to the platform language, and to the CBOR library you rely on (more details in the encoding section).

**ConnectionData**:
```go
type ConnectionData struct {
	Addr: string // lowercased MAC address e.g. "d8:12:65:b4:13:c5"
	Key:  string // 32 bytes, lowercase hex encoded
}
```

**ByteString**:
```go
// As encoding raw byte arrays to CBOR is not handled very well by
// most libraries out there, we encapsulate those in a one-field
// structure.
type Bytestring struct {
	Bytes []byte
}
```

**EnrollData**:
```go
// EnrollData contains the TPM's endorsement RSA public key
// with an optional certificate.
// It also contains a boolean @PCRExtend that indicates the new verifier
// must generate a new secret to send back on attestation success.
type EnrollData struct {
	EKCert    []byte // x509 key certificate (one byte set to 0 if none)
	EKPub     []byte // Raw public key bytes
	EKExp     int    // Public key exponent
	PCRExtend bool   // PCR extension flag
}
```

**AttestationParameters**: [attest package](https://pkg.go.dev/github.com/google/go-attestation/attest?utm_source=godoc#AttestationParameters)
```go
type AttestationParameters struct {
    // Public represents the AK's canonical encoding. This blob includes the
  	// public key, as well as signing parameters such as the hash algorithm
    // used to generate quotes.
    // Use ParseAKPublic to access the key's data.
    Public []byte

    // UseTCSDActivationFormat is set when tcsd (trousers daemon) is operating
    // as an intermediary between this library and the TPM. A value of true
    // indicates that activation challenges should use the TCSD-specific format.
    UseTCSDActivationFormat bool

    // CreateData represents the properties of a TPM 2.0 key. It is encoded
    // as a TPMS_CREATION_DATA structure.
    CreateData []byte
    // CreateAttestation represents an assertion as to the details of the key.
    // It is encoded as a TPMS_ATTEST structure.
    CreateAttestation []byte
    // CreateSignature represents a signature of the CreateAttestation structure.
    // It is encoded as a TPMT_SIGNATURE structure.
    CreateSignature []byte
}
```

**EncryptedCredentials**: [attest package](https://pkg.go.dev/github.com/google/go-attestation/attest?utm_source=godoc#EncryptedCredential)
```go
type EncryptedCredential struct {
    Credential []byte
    Secret     []byte
}
```

**PlatformParameters**: [attest package](https://pkg.go.dev/github.com/google/go-attestation/attest?utm_source=godoc#PlatformParameters)
```go
type PlatformParameters struct {
    // The version of the TPM which generated this attestation.
    TPMVersion TPMVersion
    // The public blob of the AK which endorsed the platform state. This can
    // be decoded to verify the adjacent quotes using ParseAKPublic().
    Public []byte
    // The set of quotes which endorse the state of the PCRs.
    Quotes []Quote
    // The set of expected PCR values, which are used in replaying the event log
    // to verify digests were not tampered with.
    PCRs []PCR
    // The raw event log provided by the platform. This can be processed with
    // ParseEventLog().
    EventLog []byte
}
```

**AttestationResponse**:
```go
type AttestationResponse struct {
    Err      bool
    Response []byte
}
```

Note that structures took from the go-attestation packge contains non-standard types fields. You'll also need to reimplement those inner types in order to be able to marshal/unmarshal CBOR.
Another options is to use the go-attestation package on client side, which is the approach of currently available clients.

# Encoding

During the protocol, messages are sent in an encoded way. This section describes how they are.
As described earlier, there are three types of message encoding in this protocol: JSON/CBOR/CBOR+AES.
You'll also need to take care of Bluetooth Low Energy encoding, described in the last subsection.

## JSON:

The only message encoded to JSON in the protocol is the one present in the QR code. It aims to give the client information on how to connect to the server, and to securely give it the AES symmetric key. It is formatted as follow:

```json
{"Addr":"xx:xx:xx:xx:xx:xx","Key":"00112233445566778899aabbccddeeff00112233445566778899aabbccddeeff"}
```

## CBOR:

Every message of the Ultrablue protocol that goes over BLE is first [CBOR](https://cbor.io) encoded.
CBOR librairies often provides functions/methods like `Marshal` and `Unmarshal` to go get a binary object from structured data.
The server expects the client to send it compatible encoded objects. You'll probably need to read the documentation of the CBOR library you use to find out.

Anyway, this results in simple binary blobs like this:

```
0                   CBOR size
+-+-+-+-+-+-+-+-+-+-+
|     CBOR data     |
+-+-+-+-+-+-+-+-+-+-+
```

## CBOR + AES

Most of the messages are encrypted, thus adding another level of encoding. The chosen encryption algorithm is AES/GCM with a key length of 256 bits, and no padding. Each message is encrypted using a ranodm IV of 12 bytes, which prefixes the message. The receiver is able to decrypt the message with the prepended IV and the secret key it knows.

Encrypted messages looks like this:
```
0           12                            12 + CBOR size
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|     IV    |     encrypted CBOR data     |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

## BLE:

Bluetooth Low Energy is designed to send very small messages (often less than 10 bytes), but Ultrablue uses it to carry information up to 40K bytes. As BLE doesn't support such messages, we need to split messages into several packets. The maximum size of a packet is defined by the MTU. As a consequence, the receiver needs to know how long is the incoming message, to be able to reconstruct it when fully received. For this reason, each message that goes over BLE is prefixed by its size, on four bytes, little endian encoded. If the message is carried over several packets, only the first packet is prefixed with the size of the full message:

Here is for example a message of 42 bytes long, encoded/encrypted, sent with a MTU of 20

```
Packet n.1:

0                 4                             20
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|   Message size  |        encoded[0:16]        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

Packet n.2:

0                                              20
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                encoded[16:36]                 |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

Packet n.3:

0                       6
+-+-+-+-+-+-+-+-+-+-+-+-+
|     encoded[36:42]    |
+-+-+-+-+-+-+-+-+-+-+-+-+
```

For now, the client doesn't have to send such big packets, so if you use a big enough MTU (512 is recommended), you'll not need to chunk the messages you send. However, you still need to prefix your messages with the size field, and you'll also need to reconstruct the server messages.
