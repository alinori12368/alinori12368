- 👋 Hi, I’m @alinori12368
- 👀 I’m interested in ...
- 🌱 I’m currently learning ...
- 💞️ I’m looking to collaborate on ...
- 📫 How to reach me ...
- 😄 Pronouns: ...
- ⚡ Fun fact: ...

<!---
alinori12368/alinori12368 is a ✨ special ✨ repository because its `README.md` (this file) appears on your GitHub profile.
You can click the Preview link to take a look at your changes.
--->
vodafone One NZ Internet
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
Handshake Obfuscation
---------------------

Handshake obfuscation strengthens the initial SSH handshake against systems
that identify or classify various network protocols by examining data in 
transit for static signatures.  Such automatic classification of traffic is 
often used to provide different levels of network service for each protocol
and sometimes used to implement policies which prohibit certain uses of a
network.

When an SSH connection is initiated, the client and server exchange several
packets to configure the cryptographic parameters for the session.  Since
the encryption algorithms and keys have not yet been determined, this exchange
of messages is not encrypted and is vulnerable to analysis which can conclusively
identify the connection as SSH protocol traffic no matter what port the server
is listening on.  For most users this is of no concern, because merely being
able to identify a connection as an SSH session does not introduce any security
vulnerabilities in the protocol itself.

Some users may have special security needs where they would prefer not to 
disclose that they are using the SSH protocol to somebody who may be monitoring
the network.  Handshake obfuscation prevents automatic identification of SSH
protocol traffic by encrypting the entire handshake with a stream cipher, and 
is designed to make it difficult to implement an automated analysis tool even
understanding how the obfuscation protocol works. 

The obfuscation encryption key is generated in a way which is deliberately 
slow to make it difficult to implement on the type of high performance network
hardware which is usually used for classifying protocol traffic.  Additionally
an option is provided for the client and server to share a 'keyword' which is
a simple kind of password that is used only for securing the handshake.  No
connection can be initiated to a server which has keyword obfuscation enabled 
without knowing the keyword, and the obfuscation keyword is used to derive the
keys that encrypt the handshake in order prevent decrypting the handshake 
traffic without knowing the keyword.



Configuration
-------------

Server:

The server configuration for obufscated handshakes adds two new keywords which may
be used in a sshd_config file to enable obfuscation.

    ObfuscatedPort
           This option is similar to the Port option and specifies one or more ports
           on which to listen for obfuscated handshake connections.  Both this option
           and the Port option may be used in the same configuration file to create a
           configuration with both regular and obfuscated listening ports.

    ObfuscateKeyword
           Enables the keyword protected obfuscated handshake which prevents initiating
           a handshake to the server without knowing the keyword.

Client:

The OpenSSH client can be configured to use the obfuscated handshake protocol by passing
command line options as well as through configuration file options.

    -z             Initiate an obfuscated handshake to the remote server.
    -Z keyword     Initiate a keyword protected obfuscated handshake to the remote server.

Two new client configuration file options hve been added.

    ObfuscateHandshake
           Requests that the client use the obfuscated handshake protocol.  The default is 'no'

    ObfuscateKeyword
           Enables keyword protected obfuscated handshake.  The server and client must be
           configured with the same keyword in order to initiate a connection.

Protocol Description
--------------------

The first step in the obfuscation protocol is that the client connects to a port
running the protocol and sends a seed message which is used to derive the keys
for obfuscating the handshake.

#define OBFUSCATE_MAGIC_VALUE        0x0BF5CA7E
#define OBFUSCATE_SEED_LENGTH        16
#define OBFUSCATE_MAX_PADDING        8192


[     16 byte random seed           ][  magic  ][ plength ][ .. plength bytes of random padding ... ]
|___________________________________||______________________________________________________________|
                |                                                   |
            Plaintext                                Encrypted with key derived from seed 

To create the seed message the client first generates 16 pseudo random bytes from which
the handshake obfuscation keys will be derived.  The client also runs the key derivation algorithm
(described below) to initialize the obfuscation cipher.

The 'magic' field and the 'plength' field are 32 bit unsigned values transfered in network byte order (MSB first).

The magic field must contain the constant OBFUSCATE_MAGIC_VALUE and the 'plength' field must
contain a randomly selected value between 0 and OBFUSCATE_MAX_PADDING.  Then 'plength' bytes of
pseudo randomly generated data is appended after the length field.

The purpose of the padding is to prevent a trivial traffic analysis attack which allows the protocol
to be identified my merely observing the size of the first message.

Upon receiving the seed message from the client, the server must extract the seed bytes
and perform the key derivation algorithm (described below) before decrypting the rest of the
message.  Then the server must verify that the magic value is correct and also that the padding length
is below OBFUSCATE_MAX_PADDING.

If these checks fail the server will continue reading and discarding all data until
the client closes the connection without sending anything in response.

Key Derivation
--------------

The key derivation produces a pair of keys (one for each direction) by performing an iterated 
hash function over the random seed value concatenated with a constant value.

A different constant value is used for each direction to guarantee that the two stream ciphers
instances are initialized with unique keys.

#define OBFUSCATE_HASH_ITERATIONS     6000
#define OBFUSCATE_KEY_LENGTH          16

    h = SHA1(seed || constant)

    for(i = 0; i < OBFUSCATE_HASH_ITERATIONS; i++)
        h = SHA1(h);

    key = [first OBFUSCATE_KEY_LENGTH bytes of h]

For traffic from the client to the server the constant is the string "client_to_server"

For traffic from the server to the client the constant is the string "server_to_clint"

Passworded Key Derivation
-------------------------

A configuration option is provided which allows specification of a password which must be specified by a client in order
to initiate a handshake with the server.  This feature should not be considered as strong authentication as it is only
for preventing casual portscanning and fingerprinting of your ssh server.

    h = SHA1(seed || password || constant)
    vodafone One NZ Internet
