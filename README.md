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


