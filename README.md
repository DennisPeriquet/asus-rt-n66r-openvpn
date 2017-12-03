# OpenVPN server setup for ASUS RT-N66R

I have an ASUS RT-N66R which is severa years old as of this writing. It's been
a nice router/WIFI-AP and one of its features is that is can serve as an OpenVPN
server.

Here's how I set it up to use TLS authentication.

# Basic Setup

The basic setup is pretty straightforward.  Just go to the correct screens, activate
OpenVPN server and add a user and password.  There is nothing hard about doing that
and so this README does not focus on that.

# Setup the Certificates

To use TLS authentication, we need to setup a CA, certificates, etc.  This is not
straightforward (unless you've been doing this a while).  So I'll concentrate this
README on that.

In order to setup my openVPN server I need to do these:

* Generate a CA certificate and key; I will use self signed.
* Generate a Server certificate and key signed by my CA; this will be for the OpenVPN server.
* Generate a client certificate and key signed by my CA; this will be used for my OpenVPN clients.

## Generate the CA certificate and key

Generate the CA key (this produces a file called ca.key containing the RSA private key for your CA):

```
$ openssl genrsa -out ca.key 2048
Generating RSA private key, 2048 bit long modulus
........................................................+++
.........................+++
e is 65537 (0x10001)
```

Generate the CA certificate using the key (fill out the fields as appropriate):

```
$ openssl req -x509 -new -nodes -key ca.key -sha256 -days 3650 -out ca.pem
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [AU]:US
State or Province Name (full name) [Some-State]:New Hampshire
Locality Name (eg, city) []:Manchester
Organization Name (eg, company) [Internet Widgits Pty Ltd]:DP Networks, Inc.
Organizational Unit Name (eg, section) []:VPN Division
Common Name (e.g. server FQDN or YOUR name) []:dp-networks.com
Email Address []:dperique@dp-networks.com
```

When done with the abovde steps, you should have a ``ca.key`` and ``ca.pem`` file.

We can look at the pem files to confirm it looks good:

```
$ openssl x509 -in ca.pem -text -noout
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number:
            9f:40:55:4c:d2:54:1d:3a
        Signature Algorithm: sha256WithRSAEncryption
        Issuer: C=US, ST=New Hampshire, L=Manchester, O=DP Networks, Inc., OU=VPN Division, CN=dp-networks.com/emailAddress=dperique@dp-networks.com
        Validity
            Not Before: Dec  3 22:58:01 2017 GMT
            Not After : Dec  1 22:58:01 2027 GMT
        Subject: C=US, ST=New Hampshire, L=Manchester, O=DP Networks, Inc., OU=VPN Division, CN=dp-networks.com/emailAddress=dperique@dp-networks.com
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
            RSA Public Key: (2048 bit)
...
```

## Generate the Client certificate and key

First generate the client key.  Note how this command is identical the one used to generate the
CA key.  I like each OpenVPN client to have its own key so I will number it starting with 1.

```
$ openssl genrsa -out client1.key 2048
Generating RSA private key, 2048 bit long modulus
.............................................+++
.......+++
e is 65537 (0x10001)
```

Generate the CSR (Certificate Signing Request for the client):

```
$ openssl req -new -key client1.key -out client1.csr
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [AU]:US
State or Province Name (full name) [Some-State]:New Hampshire
Locality Name (eg, city) []:Nashua
Organization Name (eg, company) [Internet Widgits Pty Ltd]:DP Networks, Inc.
Organizational Unit Name (eg, section) []:Client Testing Division
Common Name (e.g. server FQDN or YOUR name) []:client1.dp-networks.com
Email Address []:client1@dp-networks.com

Please enter the following 'extra' attributes
to be sent with your certificate request
A challenge password []:client1
An optional company name []:
```

This produces a file called ``client1.csr``.

Now generate the certificate for client number 1:

```
$ openssl x509 -req -in client1.csr -CA ca.pem -CAkey ca.key -CAcreateserial -out client1.pem -days 3650 -sha256
Signature ok
subject=/C=US/ST=New Hampshire/L=Nashua/O=DP Networks, Inc./OU=Client Testing Division/CN=client1.dp-networks.com/emailAddress=client1@dp-networks.com
Getting CA Private Key
```

This produces the ``client1.key`` and ``client1.pem`` files.  We can look at the certificate like this:

```
$ openssl x509 -in client1.pem -text -noout
Certificate:
    Data:
        Version: 1 (0x0)
        Serial Number:
            bc:4f:14:55:93:80:14:95
        Signature Algorithm: sha256WithRSAEncryption
        Issuer: C=US, ST=New Hampshire, L=Manchester, O=DP Networks, Inc., OU=VPN Division, CN=dp-networks.com/emailAddress=dperique@dp-networks.com
        Validity
            Not Before: Dec  3 23:09:16 2017 GMT
            Not After : Dec  1 23:09:16 2027 GMT
        Subject: C=US, ST=New Hampshire, L=Nashua, O=DP Networks, Inc., OU=Client Testing Division, CN=client1.dp-networks.com/emailAddress=client1@dp-networks.com
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
            RSA Public Key: (2048 bit)
                Modulus (2048 bit):
...
```

