# Ultrablue server

This section describes the Ultrablue server, available from the ANSSI-FR repository. Only Linux is currently supported.

## Installation

You can download and compile the Ultrablue server with the following shell commands:
```sh
git clone git@github.com:ANSSI-FR/ultrablue.git
cd ultrablue/server
go get ultrablue-server
go build
```

If you want, you can also install Ultrablue for your user:
```sh
cp ultrablue-server $HOME/.local/bin/ #Make sure it is included in your PATH.
```

Or systemwise:
```sh
sudo cp ultrablue-server /usr/bin/
```

## Usage

```
sudo ultrablue-server [options]
```

Note that you need to run `ultrablue-server` as root, as it performs privileged actions with devices such as the Bluetooth controller and the TPM, and with files in `/etc/ultrablue`.

### Options:

```
--enroll:
	When used, the server will start in enroll mode, needed to register
	a new verifier with a client app.
	If this option isn't used, the server will start in attestation mode.

--loglevel:
	The loglevel flag takes an integer parameter between 0 and 3.
	It indicates the verbosity level of the server.
	0 stands for no log, 2 for maximum output.

--mtu:
	Sets the MTU (Maximum transmission Unit) size for the BLE packets.
	Must be between 20 and 500 to be effective.

--pcr-extend:
	Extends the 9th PCR with the verifier secret on attestation success.
	This is mainly used to perform disk decyption.

--with-pin:
	At enroll time, a symmetric key used to encrypt communications
	is stored in /etc/ultrablue/CLIENT_UUID*. This key is sealed to the TPM
	Storage Root Key, so that it can't be decrypted on another computer.
	Using the --with-pin option will add another layer of security,
	also sealing the key with a policy password. This flag must either be
	specified both at enroll time and for an attestation, or never used for
	a specific client.
```

## Testing

```
# GOTMPDIR is only necessary if /tmp is set as noexec.
GOTMPDIR=$XDG_RUNTIME_DIR go test
```

There is also a [testbed to do end-to-end testing](testbed/) on virtual machines.
