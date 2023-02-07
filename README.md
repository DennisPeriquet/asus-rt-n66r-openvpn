# OpenVPN server setup for ASUS RT-N66R

I have an ASUS RT-N66R which is several years old as of this writing. It's been
a nice router/WIFI-AP and one of its features is that is can serve as an OpenVPN
server.

Here's how I set it up to use TLS authentication and using self signed certificates.

This was done on a Mac using the open source openssl tool but should be very similar
on Linux machines.

# Basic Setup

The basic setup for OpenVPN on the ASIS router is pretty straightforward.  Just
go to Advanced Settings, VPN, VPN Server, OpenVPN. Activate OpenVPN server and
add a user/password.  There is nothing difficult about doing that and so this
README does not focus on that.

# Setup the Certificates

To use TLS authentication, we need to setup a CA, certificates, etc.  This is not
straightforward (unless you've been doing this a while).  So I'll concentrate this
README on that.

In order to setup my openVPN server I need to generate:

* a CA certificate and key; I will use self signed.
* a client certificate and key signed by my CA; this will be used for my OpenVPN clients.
* a Server certificate and key signed by my CA; this will be for the OpenVPN server.

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

When done with the above steps, you should have a ``ca.key`` and ``ca.pem`` file.

We can look at the pem file to confirm it looks good:

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

## Generate the Client key and certificate

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
A challenge password []:client1 <-- you can make this as strong as you want
An optional company name []:
```

This produces a file called ``client1.csr``.

Now generate the certificate for client number 1 using the key and client1 CSR just created.

```
$ openssl x509 -req -in client1.csr -CA ca.pem -CAkey ca.key -CAcreateserial -out client1.pem -days 3650 -sha256
Signature ok
subject=/C=US/ST=New Hampshire/L=Nashua/O=DP Networks, Inc./OU=Client Testing Division/CN=client1.dp-networks.com/emailAddress=client1@dp-networks.com
Getting CA Private Key
```

You should now have the ``client1.key`` and ``client1.pem`` files.

We can look at the certificate  (client1.pem) to confirm it looks good like this:

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

## Generate the Server key and certificate

Essentially, we repeat the process we did for client number 1 above.

Generate a key for the Server.

```
$ openssl genrsa -out server.key 2048
Generating RSA private key, 2048 bit long modulus
...........................+++
...................................................................................................................+++
e is 65537 (0x10001)
```

Generate a CSR for the Server:

```
$ openssl req -new -key server.key -out server.csr
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
Organizational Unit Name (eg, section) []:Server Division
Common Name (e.g. server FQDN or YOUR name) []:server.dp-networks.com
Email Address []:server@dp-networks.com

Please enter the following 'extra' attributes
to be sent with your certificate request
A challenge password []:server    <-- you can make this as strong as you want
An optional company name []:
```

Generate the certificate for the Server using the server CSR and key:

```
$ openssl x509 -req -in server.csr -CA ca.pem -CAkey ca.key -CAcreateserial -out server.pem -days 3650 -sha256
Signature ok
subject=/C=US/ST=New Hampshire/L=Manchester/O=DP Networks, Inc./OU=Server Division/CN=server.dp-networks.com/emailAddress=server@dp-networks.com
Getting CA Private Key
```

# Configure the ASUS router with the certificates just created

Once you have certificates, you can now configure them on the OpenVPN server and client.

## Configure the OpenVPN server with the Server certificate

On the ASUS router, navigate to Advanced Settings, VPN, VPN Server, OpenVPN.  At VPN Details,
select Advanced Settings.  At Authorization Mode, select TLS.  Then click on the link that says
"Content modification of Keys & Certification".

Here, you will see these fields.  For each one, remove what is currently (if there is anything
there) and fill in with the indicated contents:

* Certificate Authority: paste the contents of ``ca.pem``
* Server Certificate: paste the contents of ``server.pem``
* Server Key: paste the contents of ``server.key``
* Diffie Hellman parameters: leave what is there (defaults)
* Certificate Revocation List (Optional): leave blank

Click Save to save what you just pasted.

Click Apply to apply these settings.

## Configure the OpenVPN client with the Client certificate

At the VPN Details on the ASUS router UI, select General.  Click EXPORT.

You will get a file called ``client.ovpn``.

Edit the ``client.ovpn`` file.

Change the line that says ``remote x.x.x.x 1194`` so that the x.x.x.x is the IP address of your
OpenVPN server reachable from the OpenVPN clients.  This could be the Public IP address of your
cable modem and might need to be configured with port forwarding for port 1194.

Comment out the ``ns-cert-type server``.  Do this by putting a semi-colon before it
like ``; ns-cert-type server``.

For the next two steps, N=1.  If you created more client cert/key pairs, increment N.

In the section marked by ``<cert>`` and ``</cert>``, paste the contents of the ``clientN.pem`` file.

In the section marked by ``<key>`` and ``</key>``, paste the contents of the ``clientN.key`` file.

Save the file.  Add this file into your OpenVPN client (how to do that depends on your OpenVPN
client.  On MacOS, you just drag it into the OpenVPN client UI.

# Test your OpenVPN client

You should be able to get onto your VPN by activating the VPN client and entering your user
credentials.  If not you will have troubleshoot.

# Summary

The above is summarized below to make creating certs and keys faster to be conducive to
frequent rotations.

TIP: save your challenge password and credentials in a reputable password manager.

```
# Generate ca key
openssl genrsa -out ca.key 2048

# Generate the ca cert
openssl req -x509 -new -nodes -key ca.key -sha256 -days 3650 -out ca.pem

# View the ca cert
openssl x509 -in ca.pem -text -noout

# Generate key for client1
openssl genrsa -out client1.key 2048

# Generate csr for client1
openssl req -new -key client1.key -out client1.csr

# Generate the cert for client 1 using the client1 csr
openssl x509 -req -in client1.csr -CA ca.pem -CAkey ca.key -CAcreateserial -out client1.pem -days 3650 -sha256

# View the client1 cert
openssl x509 -in client1.pem -text -noout

# Generate the server key
openssl genrsa -out server.key 2048

# Generate a csr for the server
openssl req -new -key server.key -out server.csr

# Generate the cert for server using the server csr
openssl x509 -req -in server.csr -CA ca.pem -CAkey ca.key -CAcreateserial -out server.pem -days 3650 -sha256
```

Configure your router with the newly generated ca cert and server cert/key
To generate for client N:

```
export N=2
# Generate key for client${N}
openssl genrsa -out client${N}.key 2048

# Generate csr for clientN
openssl req -new -key client${N}.key -out client${N}.csr

# Generate the cert for client N using the clientN csr
openssl x509 -req -in client${N}.csr -CA ca.pem -CAkey ca.key -CAcreateserial -out client${N}.pem -days 3650 -sha256

# Copy the existing config so you can tweak it
cp home-vpn1.ovpn home-vpn2.ovpn
vi home-vpn2.ovpn
  insert the client2.pem between <cert> and </cert>
  insert the client2.key between <key> and </key>
```

