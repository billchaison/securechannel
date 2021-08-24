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

The domain controller sends an encrypted server credential back to the workstation to confirm that it knows the current machine password as well.  The algorithm is identical to the request made by the workstation in figure 3, except that the server challenge `d080554d6ac171eb` is substituted in step 4.

![alt text](https://github.com/billchaison/securechannel/blob/main/sc06.png)

Compute the encrypted server credential using AES-128-CFB8 and the session key on the server challenge.<br />
`echo d080554d6ac171eb | xxd -r -p | openssl enc -aes-128-cfb8 -K 3bde5fef94210c211159c347e10c0189 -iv 00000000000000000000000000000000 | xxd -p`<br />
The output is the server credential `5873518535259fdd` seen in the packet capture.  The workstation is able to recompute this value to validate the domain controller by using the private session key and challenge nonces.

**Decrypting subsequent packets**

After mutual authentication has completed successfully, additional steps in the protocol take place.  The following two captured frames show the NetrLogonGetDomainInfo operation.  If the frames contain a Sign algorithm value of `0x0013` and a Seal algorithm value of `0x001a` then the AES-128-CFB8 cipher is used again to encrypt the stub data.

**(5) NetrLogonGetDomainInfo request**

The workstation initiates the request.  Note the Sequence No `6e9e03a812b04528`, which is encrypted, and the Packet Digest `0802f92282a7474c`.  These will be used to decrypt the stub data.

![alt text](https://github.com/billchaison/securechannel/blob/main/sc07.png)

1. Decrypt the encrypted sequence number `6e9e03a812b04528` by first taking the Packet Digest and concatenating it to itself to form the IV `0802f92282a7474c0802f92282a7474c`<br />
2. Using the session key computed earlier `3bde5fef94210c211159c347e10c0189` decrypt the sequence number using the following bash command pipeline.<br />
`echo 6e9e03a812b04528 | xxd -r -p | openssl enc -aes-128-cfb8 -K 3bde5fef94210c211159c347e10c0189 -iv 0802f92282a7474c0802f92282a7474c -d | xxd -p`<br />
The output is the decrypted sequence number `0000000280000000`<br />
3. Concatenate the sequence number to itself to get the IV that will be used to decrypt the stub data `00000002800000000000000280000000`<br />
4. Derive a decryption key by taking the session key and XORing every byte with `0xf0`.  The following bash command will accomplish this.<br />
`printf "%16x%16x\n" $((0x3bde5fef94210c21^0xf0f0f0f0f0f0f0f0)) $((0x1159c347e10c0189^0xf0f0f0f0f0f0f0f0))`
The output is the decryption key `cb2eaf1f64d1fcd1e1a933b711fcf179`<br />
5. The encrypted stub data in this frame is as follows:<br />
```
c5dfe9e103ecbf96fcf35b550d7a6a9c97b38d55d68dbee09caba761c635e144
d1610a309e674bb825e53f6f9a869ef4aa7afc1f77b3bab49bec0b2173ef4698
f6a3a3d0f57d28524d91cda97663b8ef4d838be76c7e85b694d9e60f7cee4c20
8a8ce3e60b3cce79bfe5b9500ee955c4c89e24acbef070f5ab48cf2ed2141af1
a3a996a2fcbc43434ebaffa9adbc65324b41c92e6153443c06a45e74d1c876fb
16bfaee301336966f83d9a3e5929f60054fc8b1ef93f1840154c5b79643e5cf1
045a6dd9a84ea879eb93c79fa7bfc200fbe35b49996fe44c452b0ac6841538e4
1f2abe224a722e0f2f8c8283b9a9232cedaac1b3f95a7cc1f78da805f3b35e84
40f813cf5ed7539f89051993858c71c4775fc1b5b01b10772734d2e99b3559cb
574c049de0db1c28bb698ea8dabf765fb587058018018e9254430268c1d9d198
c9a68b845f92477a49032c8b33dfd4886eacbdbbac173ceeade4cd1b10b3e7cd
fdff5d526592541217bdd4ea72eab3c073949239a2cc7c2ce976559098795044
f9fa614f1e7d38417e93472450af7590dcc14c4b2e2896c959e530990ffa83f9
e74ef3154d6263f95bd128c2d98ec66debc33913f14573271f824c30ccc41b83
5552fc64afdbeda435e22e7d5dae84979f12aacd2a43aec3ac48174a65897c28
6181ffcc813a3a3a2e005ca77c62cfab3f8d4856f6c7639e8eb59d4610263772
8588482f05045af0925d85f8acd02413440e255eac593c1558f87564ca8cbcef
5fb26dc2a812f1ac36e4d7c281c0834b14264199b1fe9089d965534c929cdc82
2db015a29424e6e9ef2cca612c3b9561f90b11c1bb504ea2ee75a1053b1a46c0
81831310f6db637cdb4bf1d7bff4cc15fae23de7cff9033f99b8252db9835031
5aa9ac70605717b346ba3e5b376efb3bd406737eae66c1492b6b57cfa81cc8bf
f8e7cd1c822820f56a5611a4cbd547ee8259a601100fae4f9be98c05df1fff63
8471641a87c081a13d28dae7505ff48069841ea689dfcdc4e8ea55d504219862
8350f5d70ed7ee81b2488fbf4d2f89ec0338ebe398ffab6ac5515d8feaddec15
04746e4181d6c6d6f5089d737dd949fa2947705a3bdca0c687471b20ae72721d
caccc39fc7f18512c5ea7387482ea76dd799f5fa6d0e78917d4ca602820b269a
```
6. Decrypt the stub data using the IV and key from steps 3 and 4 using the following bash command pipeline to extract the unicode strings.<br />
`echo c5dfe9e103ecbf96fcf35b550d7a6a9c97b38d55d68dbee09caba761c635e144d1610a309e674bb825e53f6f9a869ef4aa7afc1f77b3bab49bec0b2173ef4698f6a3a3d0f57d28524d91cda97663b8ef4d838be76c7e85b694d9e60f7cee4c208a8ce3e60b3cce79bfe5b9500ee955c4c89e24acbef070f5ab48cf2ed2141af1a3a996a2fcbc43434ebaffa9adbc65324b41c92e6153443c06a45e74d1c876fb16bfaee301336966f83d9a3e5929f60054fc8b1ef93f1840154c5b79643e5cf1045a6dd9a84ea879eb93c79fa7bfc200fbe35b49996fe44c452b0ac6841538e41f2abe224a722e0f2f8c8283b9a9232cedaac1b3f95a7cc1f78da805f3b35e8440f813cf5ed7539f89051993858c71c4775fc1b5b01b10772734d2e99b3559cb574c049de0db1c28bb698ea8dabf765fb587058018018e9254430268c1d9d198c9a68b845f92477a49032c8b33dfd4886eacbdbbac173ceeade4cd1b10b3e7cdfdff5d526592541217bdd4ea72eab3c073949239a2cc7c2ce976559098795044f9fa614f1e7d38417e93472450af7590dcc14c4b2e2896c959e530990ffa83f9e74ef3154d6263f95bd128c2d98ec66debc33913f14573271f824c30ccc41b835552fc64afdbeda435e22e7d5dae84979f12aacd2a43aec3ac48174a65897c286181ffcc813a3a3a2e005ca77c62cfab3f8d4856f6c7639e8eb59d46102637728588482f05045af0925d85f8acd02413440e255eac593c1558f87564ca8cbcef5fb26dc2a812f1ac36e4d7c281c0834b14264199b1fe9089d965534c929cdc822db015a29424e6e9ef2cca612c3b9561f90b11c1bb504ea2ee75a1053b1a46c081831310f6db637cdb4bf1d7bff4cc15fae23de7cff9033f99b8252db98350315aa9ac70605717b346ba3e5b376efb3bd406737eae66c1492b6b57cfa81cc8bff8e7cd1c822820f56a5611a4cbd547ee8259a601100fae4f9be98c05df1fff638471641a87c081a13d28dae7505ff48069841ea689dfcdc4e8ea55d5042198628350f5d70ed7ee81b2488fbf4d2f89ec0338ebe398ffab6ac5515d8feaddec1504746e4181d6c6d6f5089d737dd949fa2947705a3bdca0c687471b20ae72721dcaccc39fc7f18512c5ea7387482ea76dd799f5fa6d0e78917d4ca602820b269a | xxd -r -p | openssl enc -aes-128-cfb8 -K cb2eaf1f64d1fcd1e1a933b711fcf179 -iv 00000002800000000000000280000000 -d | strings -e l`<br />
The output will look like this:<br />
```
\\WIN2019DC.DOMLAB.NET
WIN10WKSTN
WIN10WKSTN.DOMLAB.NET
Default-First-Site-Name
Windows 10 Pro
```

**(6) NetrLogonGetDomainInfo response**

The domain controller responds to the workstation.  Note the Sequence No `87ff42dcf095994e`, which is encrypted, and the Packet Digest `113ac37e4f4cb72d`.  These will be used to decrypt the stub data.

![alt text](https://github.com/billchaison/securechannel/blob/main/sc08.png)

The decryption algorithm is identical to what appears in figure 5, except that the sequence number, packet digest and stub data from this frame are substituted.

1. IV is the Packet Digest concatenated to itself `113ac37e4f4cb72d113ac37e4f4cb72d`<br />
2. Using the session key computed earlier `3bde5fef94210c211159c347e10c0189` decrypt the sequence number using the following bash command pipeline.<br />
`echo 87ff42dcf095994e | xxd -r -p | openssl enc -aes-128-cfb8 -K 3bde5fef94210c211159c347e10c0189 -iv 113ac37e4f4cb72d113ac37e4f4cb72d -d | xxd -p`<br />
The output is the decrypted sequence number `0000000300000000`<br />
3. Concatenate the sequence number to itself to get the IV that will be used to decrypt the stub data `00000003000000000000000300000000`<br />
4. Derive a decryption key by taking the session key and XORing every byte with `0xf0`.  The following bash command will accomplish this.<br />
`printf "%16x%16x\n" $((0x3bde5fef94210c21^0xf0f0f0f0f0f0f0f0)) $((0x1159c347e10c0189^0xf0f0f0f0f0f0f0f0))`
The output is the decryption key `cb2eaf1f64d1fcd1e1a933b711fcf179`<br />
5. The encrypted stub data in this frame is as follows:<br />
```
a4d9e4683f6f9d34377a54ce23fb5f7ae991b24ee6c5aaadeef08039a9ce7dd8
f70f923614327d8656e27c9505b6da007e7dc51316937ed774a439530697381b
7c827c291df40fbd9b374bc69a825492b914f6a79a861a7ae78b0bf58036b6bf
07e8f1b18043a910ea3d14f4eb3a52d340ac2f3ccc8e46c595215f11c786242d
63e4e2e765330fbb655b8b154aad6d3aacc41d2c6a368628b3c341c7d267ff5b
ff0d77e2c75d06ebfbffc50926324b5771c41c0205297b9fb2d66471f8ffc71c
63e322087d5994a3826be1acd9d88246a5ddfc1960f7c1fa9249b4dfbb7788e5
0fc982ec149dc44785a12b561f4ad850ac1480f2b8fc062f80b782189111f39e
080e5e34ad59e4d0de4749119a6d66c0f8593b3b7785f3d95cf9ddfa678b7727
d8faf7a0743a561c247b7dc46a4cfac4a1ac1e99fac4a305f4b4b3d738c59e0b
dc34a2547a87a3e160c3d1bc0ba25e29121de13f5bf55e8e7153945e9c20dec2
baab44997bcee39e7e0dbe31e591785e49fdfa16bcc3954126c24b0ef15e596f
7e20c3cedb4fb4516e9eee3b136e3f1e141648d276c4e507ba81750782ad1663
60125338425b0c1ff85c5d55e893227acb532cd6f57a7512b86723432724c915
2e11d1501b61c9e62ee70ad69b7a2cfaaec5442619e8518d912cfa9c3cf52ebd
26a8707a4f822a40528d812773d40fe9fd5597ba520d0fe678dea809dc2ef2a8
2bffae9e023e690872aedffe7a0c48a606a1b0ee61bcd38cabf3e255999ca866
924cf7597e90bbd294f750f61bcbca69cc9a653b4bc366a26b4fadbfd091b062
209269089dd79b37132b2fadabd2dec2369a1ea56106fde81c33f7a7fbe4da0b
d72e09958104118e5e167f00f53001c35d8966cfc523c2e579934aabfa81d315
e9f2be21e9ccac15f5c2d120869132ad137d37e8a87674e25f61d075b4755a57
70d8f51af938913680c710899c20decca1e6e789d0c0563359b2bd767ca42120
72224b74c3c25759835addeac47aece7b577dd357d8d75eafcd6482c92c090bd
489fe08988607e55f5e3b77b18fddde7dbf72aa6c86387aef7c9008a50785812
bf7f398b8249b8e167472aa4c3876c69cf48773284ec68e7c6b38452ee0dc90e
28ab04e6ab6d32d2f2b65efe75dc17129aa5a0f628863b7cea6dd0bd00633cfe
46fc61b264fa749fc26840a1c42dc7c30294979a08800ece625573041f87e41b
```
6. Decrypt the stub data using the IV and key from steps 3 and 4 using the following bash command pipeline to extract the unicode strings.<br />
`echo a4d9e4683f6f9d34377a54ce23fb5f7ae991b24ee6c5aaadeef08039a9ce7dd8f70f923614327d8656e27c9505b6da007e7dc51316937ed774a439530697381b7c827c291df40fbd9b374bc69a825492b914f6a79a861a7ae78b0bf58036b6bf07e8f1b18043a910ea3d14f4eb3a52d340ac2f3ccc8e46c595215f11c786242d63e4e2e765330fbb655b8b154aad6d3aacc41d2c6a368628b3c341c7d267ff5bff0d77e2c75d06ebfbffc50926324b5771c41c0205297b9fb2d66471f8ffc71c63e322087d5994a3826be1acd9d88246a5ddfc1960f7c1fa9249b4dfbb7788e50fc982ec149dc44785a12b561f4ad850ac1480f2b8fc062f80b782189111f39e080e5e34ad59e4d0de4749119a6d66c0f8593b3b7785f3d95cf9ddfa678b7727d8faf7a0743a561c247b7dc46a4cfac4a1ac1e99fac4a305f4b4b3d738c59e0bdc34a2547a87a3e160c3d1bc0ba25e29121de13f5bf55e8e7153945e9c20dec2baab44997bcee39e7e0dbe31e591785e49fdfa16bcc3954126c24b0ef15e596f7e20c3cedb4fb4516e9eee3b136e3f1e141648d276c4e507ba81750782ad166360125338425b0c1ff85c5d55e893227acb532cd6f57a7512b86723432724c9152e11d1501b61c9e62ee70ad69b7a2cfaaec5442619e8518d912cfa9c3cf52ebd26a8707a4f822a40528d812773d40fe9fd5597ba520d0fe678dea809dc2ef2a82bffae9e023e690872aedffe7a0c48a606a1b0ee61bcd38cabf3e255999ca866924cf7597e90bbd294f750f61bcbca69cc9a653b4bc366a26b4fadbfd091b062209269089dd79b37132b2fadabd2dec2369a1ea56106fde81c33f7a7fbe4da0bd72e09958104118e5e167f00f53001c35d8966cfc523c2e579934aabfa81d315e9f2be21e9ccac15f5c2d120869132ad137d37e8a87674e25f61d075b4755a5770d8f51af938913680c710899c20decca1e6e789d0c0563359b2bd767ca4212072224b74c3c25759835addeac47aece7b577dd357d8d75eafcd6482c92c090bd489fe08988607e55f5e3b77b18fddde7dbf72aa6c86387aef7c9008a50785812bf7f398b8249b8e167472aa4c3876c69cf48773284ec68e7c6b38452ee0dc90e28ab04e6ab6d32d2f2b65efe75dc17129aa5a0f628863b7cea6dd0bd00633cfe46fc61b264fa749fc26840a1c42dc7c30294979a08800ece625573041f87e41b | xxd -r -p | openssl enc -aes-128-cfb8 -K cb2eaf1f64d1fcd1e1a933b711fcf179 -iv 00000003000000000000000300000000 -d | strings -e l`<br />
The output will look like this:<br />
```
DOMLAB
DOMLAB.NET.
DOMLAB.NET.
DOMLAB
DOMLAB.NET
WIN10WKSTN.DOMLAB.NET
```
