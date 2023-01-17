
# Protocol

## Messages overview:

The following diagram represents the Ultrablue Protocol. It reads from top to bottom. Each arrow represents a message between the server (as the attester) and the client (as the verifier):

![](https://github.com/lfalkau/ultrablue/blob/client-doc/doc/implementing-clients/protocol.svg)

- Grey arrow are messages displayed on a QR code. it must be scanned to get its content.
- Blue arrows are messages sent over Bluetooth Low Energy (BLE), through the Ultrablue characteristic.

Each arrow comes with a short description of the message, followed by its datatype (in bold text). The color of the description gives information on the message encoding:
- Grey text is JSON encoded
- Black text is CBOR encoded
- Gold text is CBOR encoded, then encrypted with AES256/GCM.

Messages in the green box are only sent when performing an enrollment.

## Technical considerations:

Here are some notes on important things to know when implementing a client, in no particular order:

- **The client UUID**: It must be generated randomly at enroll time before being sent to the server. During later attestations, the same UUID needs to be sent by the client, to allow the server to fetch the shared key. If the server receives an UUID it already "knows" during an enrollment, it will fail and disconnect the client. Appropriate error handling must be done on client side (e.g. reconnect and generate a new UUID, tell the user why it failed and ask them to restart the enrollment...).

- **Encryption**: As described in the encoding section, most of messages are encrypted. When they are, the IV prefixes the encrypted message. IV must never be reused with the same key. It must be randomly generated.

- **Derived auth nonce**: To avoid usage of heavy TPM calls, an authentication process takes place at the beginning of the protocol. The server generates a random nonce, which is 16 bytes long and sends it to the client. The client (assuming it owns the shared key) decrypts the nonce, tweaks it in a way I'll describe below, re-encrypts the result and sends it to the server. The nonce tweak is needed before being sent back because else, nothing would avoid a client to send back the exact same message it got. It would be the same nonce, encrypted with the right key and a valid IV.  The chosen tweak is to split the nonce in two halfs, 8 bytes each, and to invert those. In python, this would give: `tweaked = nonce[8:] + nonce[:8]`.
This authentication round-trip takes around 180ms to be performed. TPM key generation, that could be used without freely without it, takes 3 seconds in average, but can sometimes take more than 10 seconds (Those benchmarks were done on my personal computer and you may observe different values).

- **Credential activation**: It should be obvious, but when sending the MakeCredential challenge to the attester's TPM, the nonce must be randomly generated, and when the challenge response comes back, the client must ensure it matches the generated one. This step proves to the verifier (you) that the Attestation Key you got has been generated on the same physical TPM than the one you enrolled. Relying on external TPM libraries can be helpful to avoid doing things wrong.

// TODO: Merge the credential activation and the anti replay nonce

- **Attestation process**: You will get several things from the server at this stage: 
	- A quote, containing the set of PCRs final hashes, signed with the Attestation Key previously received, and the decrypted nonce you sent during the credential activation. Both the signature and the nonce must be verified to assert the quote is genuine and to avoid replay attacks.
	- The event log, which is a file containing information on each PCR extension that happened during the boot process. This event log is not protected in any way and can have been tampered with. To be trusted, you'll first need to replay it, and assert that obtained each PCR final hash match the corresponding one in the quote (which is signed). Once done, you can use the event log with more confidence.

	Note that this is the stage where you can apply a PCR policy (choose which PCR are checked), and when to return an error, or not. This policy can be user defined or hardcoded, but if it is user defined, the client should warn the user and explain them what they are toggling. In the same way, the client should provide a report of what can have gone wrong if the policy is not satisfied after the attestation.

- **Attestation response**: Ultrablue can be used to do disk encryption. At enroll time, a flag is sent by the server to indicate weither you must send back a secret or not on attestation success. If you have to, this secret must be securely generated and securely stored. You'll need to send this secret back on later attestations for this attester.

- **Storing server devices**: To perform remote attestation and act as a verifier, data about the attester must be stored, including:
	- The attester' TPM endorsement key
	- Connection information about the attester (MAC address, advertised name...)
	- The UUID you generated at enroll time
	- The secret you generated at enroll time if you did
	- The shared symmetric key used to encrypt the communication

	Secrets should be securely stored, e.g. in the keystore on Android, in the keychain on IOS...
	// TODO: Find the best moment to store the attester object


# Data types

Here is the list of data structures used by the protocol. Some are ours, others are from the go-attestation package. Structures are written in go here, but when implementing a client, you may need to adapt them, according to the platform language, and to the CBOR library you rely on (more details in the encoding section).

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
	PCRExtend bool   // Whether or not PCR_EXTENSION_INDEX must be extended on attestation success
}
```

**AttestationParameters**: (attest package)
```go
type AttestationParameters struct {
    // Public represents the AK's canonical encoding. This blob includes the
  	// public key, as well as signing parameters such as the hash algorithm
    // used to generate quotes.
    //
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
**EncryptedCredentials**: (attest package)

```go
type EncryptedCredential struct {
    Credential []byte
    Secret     []byte
}

```
**PlatformParameters**: (attest package)
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

Note that structures took from the go-attestation packge contains non standard types fields. You'll also need to reimplement those inner types in order to be able to marshal/unmarshal CBOR.
Another options is to use the go-attestation package on client side, which is the approach of currently available clients.

# Encoding

During the protocol, messages are sent in an encoded way. This section describes how they are.
As described earlier, there are three types of message encoding in this protocol:
- JSON
- CBOR
- CBOR + AES

You'll also need to take care of Bluetooth Low Energy encoding as described in the last section.

## JSON:

The only message encoded to JSON in the protocol is the one present in the QR code. It aims to give the client information on how to connect to the server (through its MAC address), and to securely give the AES symmetric key to the client. It is formatted as follow:
```json
{"Addr":"xx:xx:xx:xx:xx:xx","Key":"00112233445566778899aabbccddeeff00112233445566778899aabbccddeeff"}
```
This message is off the network and doesn't need any other layer of encoding.

## CBOR:

Every message that goes over BLE is first CBOR encoded. CBOR means Concise Binary Object Representation and is designed to exchange data over the network.
CBOR librairies often provides functions/methods like `Marshal` and `Unmarshal` to go get a binary object from structured data.
The server expects the client to send it compatible (see the data types section) objects, CBOR encoded.

Anyway, this results in simple binary blobs like this:

```
0                   CBOR size
+-+-+-+-+-+-+-+-+-+-+
|     CBOR data     |
+-+-+-+-+-+-+-+-+-+-+
```

## CBOR + AES

Most of the messages are encrypted, adding a level of encoding. The chosen encryption algorithm is AES GCM with a key length of 256 bits, and no padding. Each message is encrypted using a ranodm IV of 12 bytes, which prefixes the message. The receiver is able to decrypt the message with the prepended IV and the secret key they know.

Encrypted messages looks like this:
```
0           12                            12 + CBOR size
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|     IV    |     encrypted CBOR data     |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

## BLE:

Bluetooth Low Energy is designed to send very small messages (often less than 10 bytes), but Ultrablue uses it to carry information up to 40K bytes. As BLE doesn't support such messages, we need to split messages into several packets. The maximum size of a packet is defined by the MTU. As a consequence, the receiver needs to know how long is the incoming message, to be able to reconstruct it when fully received. For this reason, each message that goes over BLE is prefixed by their size, on four bytes, little endian encoded. If the message is carried over several packets, only the first packet is prefixed with the size of the full message:

Here is for example a message of 42 bytes long, encoded/encrypted, sent with a MTU of 20:
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
