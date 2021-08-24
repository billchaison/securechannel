# securechannel
Step by step manual decryption of netlogon secure channel

This document describes how to manually decrypt Windows RPC Netlogon packets captured from secure channel setup between a domain-joined computer and domain controller.  The lab environment used in this write-up consists of a Windows 10 Pro workstation and Windows server 2019 domain controller.

![alt text](https://github.com/billchaison/securechannel/blob/main/sc02.png)

The Netlogon secure channel protocol follows this sequence.

![alt text](https://github.com/billchaison/securechannel/blob/main/sc01.png)

The following four captured frames show the mutual authentication portion of secure channel setup.  Secure channel is initiated by the domain-joined computer.  Challenge nonces are exchanged by both the workstation and domain controller, followed by encrypted credentials that prove both parties have prior knowledge of the computer's password (machine NTLM hash).

**(1) NetrServerReqChallenge request**

The workstation sends a randomly chosen 8-byte challenge to the domain controller and expects to receive a different 8-byte challenge in return.  The client challenge in this example is `b1d62e009b7be395`.

![alt text](https://github.com/billchaison/securechannel/blob/main/sc03.png)

**(2) NetrServerReqChallenge response**

The domain controller responds with its 8-byte server challenge back to the workstation.  The server challenge in this example is `d080554d6ac171eb`.

![alt text](https://github.com/billchaison/securechannel/blob/main/sc04.png)

**(3) NetrServerAuthenticate3 request**

The workstation sends an encrypted client credential to the domain controller to confirm that it knows the current machine password.  When the "AES Supported" flag is set, the credential is computed using the AES-128-CFB8 cipher, with a session key derived from the machine's NTLM hash and the client challenge concatenated with the server challenge.  The session key is held in memory on both the workstation and domain controller, must match, and is never seen on the network.  The session key is used to encrypt and decrypt RPC data moving forward.  The client credential in this example is `39ac09a434a5b0a3`.

![alt text](https://github.com/billchaison/securechannel/blob/main/sc05.png)

1. Concatenate the client challenge `b1d62e009b7be395` with the server challenge `d080554d6ac171eb` to form a 16-byte data vector `b1d62e009b7be395d080554d6ac171eb`.<br />
2. The machine NTLM hash for the workstation WIN10WKSTN in this example is `62d367fe7103876448561b038e91c37d`.<br />
3. Compute the session key using SHA-256 HMAC derived from the 16-byte data vector and the machine NTLM hash.  In bash use the following command pipeline to produce the session key.<br />
`echo b1d62e009b7be395d080554d6ac171eb | xxd -r -p | openssl sha256 -hex -mac HMAC -macopt hexkey:62d367fe7103876448561b038e91c37d | cut -d " " -f 2 | cut -c 1-32`<br />
The session key in this example is `3bde5fef94210c211159c347e10c0189`<br />
4. Compute the encrypted client credential using AES-128-CFB8 and the session key on the client challenge.<br />
`echo b1d62e009b7be395 | xxd -r -p | openssl enc -aes-128-cfb8 -K 3bde5fef94210c211159c347e10c0189 -iv 00000000000000000000000000000000 | xxd -p`<br />
The output is the client credential `39ac09a434a5b0a3` seen in the packet capture.  The domain controller is able to recompute this value to validate the workstation by using the private session key and challenge nonces.

**(4) NetrServerAuthenticate3 response**

The domain controller sends an encrypted server credential back to the workstation to confirm that it knows the current machine password as well.  The algorithm is identical to the request made by the workstation, except that the server challenge `d080554d6ac171eb` is substituted in step 4.

![alt text](https://github.com/billchaison/securechannel/blob/main/sc06.png)

Compute the encrypted server credential using AES-128-CFB8 and the session key on the server challenge.<br />
`echo d080554d6ac171eb | xxd -r -p | openssl enc -aes-128-cfb8 -K 3bde5fef94210c211159c347e10c0189 -iv 00000000000000000000000000000000 | xxd -p`<br />
The output is the server credential `5873518535259fdd` seen in the packet capture.  The workstation is able to recompute this value to validate the domain controller by using the private session key and challenge nonces.
