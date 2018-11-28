# BSC-OpenSSL 7.0

## Overview

BSC-OpenSSL is a variant of OpenSSl with the 
"dual-certificate" CTLS protocol and relevant features.

## Changes in 7.0

Compared to previous version, there are the following changes:

* Support CTLS protocol
* Base OpenSSL updated to 1.1.1a

## Known Issues in 7.0

* No certificate chain verification support
* No SSL trace feature for CTLS
* No RSA CTLS cipher suites support

## Introduction to CTLS

CTLS is a modified protocol based on TLSv1.1. It's standardized and published by Chinese State Cryptography Administration (SCA) via GM/T 0024-2014 specification. GM/T 0024-2014 is basically defined for SSL VPN usage, it's not exactly a standard for a generic TLS client/server scenario.

Since that GM/T 0024 is only provided in Chinese so I need to explain a bit here.

### Cipher Suites

CTLS defines 12 ciphers suites, every one of them requires different Chinese cryptography algorithm like SM2/SM3/SM4.

6 out of the 12 cipher suites requires a un-public Chinese cipher named SM1, so there's not way to implement them for a software library. 2 out of the 12 cipher suites require SM9 which has not been supported by the base OpenSSL library. So there are only 4 valid cipher suites we consider to implement in BSC-OpenSSL theoretically. They are:

* ECDHE-SM2-WITH-SM4-SM3
* ECC-SM2-WITH-SM4-SM3
* RSA-WITH-SM4-SM3
* RSA-WITH-SM4-SHA1

`ECDHE-SM2-WITH-SM4-SM3` uses SM2 key exchange algorithm in a way similar to standard ECDHE and uses SM2 signature scheme as authentication.

`ECC-SM2-WITH-SM4-SM3` uses SM2 encryption scheme to perform a key exchange process at client side and also uses SM2 signature scheme to auth.

For those RSA based cipher suites, they are a combination of RSA encryption and signature to finish the key exchange process. Since BSC-OpenSSL 7.0 doesn't support these ciphers yet, so no detail information is provided yet. Most major CTLS implementations for using in a HTTPS or a generic TLS scenario usually do not support the RSA cipher suites.

### Dual Certificates

One of the biggest changes in CTLS compared to TLSv1.1 is the `Certificate` message needs to contain 2 certificates, one for encryption and one for signature usage.

When constructing the `Certificate` - no matter for client side or server side - an implementation need to place the signature certificate first and then the encryption certificate secondly in the `Certificate` message, just like using the encryption certificate to replace the 'middle/intermedium' certificate in a chain in ordinary TLS `Certificate` message.

The GM/T 0024-2014 standard doesn't explain clear how to include the intermedium certificates for those encryption and signature certificates in `Certificate` message, which leads to we can only assume the 'middle/intermedium' certificates are not supported in CTLS - which means the root CA should always directly sign the endpoint certificates.

### Extensions

GM/T 0024-2014 says nothing about TLS extension. We assume there should be no TLS extension supported in CTLS when using with the SSL VPN scenario. For the CTLS cipher suites, all parameters are fixed for both sides who are negotiating a session. For instance, if using SM2 algorithm, the `signature algorithm` and curve stuffs are fixed with the SM2 specification, so there is no need to choose the `shared curve` or `signature algorithm` by sending TLS extensions in `Hello` messages.

One TLS extension might be an exception, the `ServerNameIndication` extension. In HTTPS or other generic TLS scenario, `SNI` in very important for the server to choose the right configuration. Since there is no clear specifications on how to address `SNI` in CTLS, we can only follow other major implementation of CTLS to adapt to them.

### Handshake

The handshake processes are different for each supported cipher suites.

#### `ECC-SM2-WITH-SM4-SM3`

For this cipher suite, it looks like a combination of pure RSA key exchange and RSA signing. It performs as follows:

1. client sends `ClientHello`
2. server sends `ServerHello`
3. server sends `Certificate` containing both signature and encryption certificates
4. server sends `ServerKeyExchange` containing the signature of `<client_random, server_random, encryption_certificate>`, which is signed by the server signature certificate(private key)
5. server sends `ServerHelloDone`
6. client stores the server signature and encryption certificates
7. client verifies the signature in `ServerKeyExchange` by using serve signature certificate
8. client sends `ClientKeyExchange` containing the encrypted `PreMasterSecret` which is encrypted by the server encryption certificate
9. client generates the symmetric key material and sends `ChangeCipherSpec` and `Finished`
10. server decrypts the `PreMasterSecret` in `ClientKeyExchange` and also generates the symmetric key material and sends `ChangeCipherSpec` and `Finished`

#### `ECDHE-SM2-WITH-SM4-SM3`

For this cipher suite, it requires client also 2 certificates - the same as what server needs to prepare - one signature certificate and one encryption certificate.

1. client sends `ClientHello`
2. server sends `ServerHello`
3. server sends `Certificate` containing both server signature and encryption certificates
4. server sends `ServerKeyExchange` containing the SM2 elliptic curve parameters and the signature of `<client_random, server_random, SM2_parameters>`. The parameters here contains curve params and the ephemeral EC public point.
5. server sends `Certificate Request`
6. server sends `ServerHelloDone`
7. client stores the server signature and encryption certificates
8. client sends `Certificate` containing both client signature and encryption certificates
8. client verifies `ServerKeyExchange` and saves the EC public point.
9. client generates a pair of EC ephemeral key and sends the EC params and the public point in `ClientKeyExchange`
10. client sends `CertificateVerify` containing the signature created by client's signature certificate
10. client derives the `MasterSecret` via SM2 key exchange algorithm with the server encryption certificate and server ephemeral public EC key.
11. client sends `ChangeCipherSpec` and `Finished`
11. server derives the `MasterSecret` via SM2 key exchange algorithm with the client encryption certificate and server ephemeral public EC key.
12. server sends `ChangeCipherSpec` and `Finished`

### Major Implementations

Currently most CTLS implementation are in hardware, for instance a 'gateway' device which can work as the CTLS offload endpoint.

Public software implementations are not common in China, but there are still some:

#### Client Side

* Qihoo 360 Browser, the special edition for CTLS

#### Server Side

* TASSL, based on OpenSSL 1.0.2. This is the most robust and production ready CTLS library you can find on the Internet. TASSL is compatible with BSC-OpenSSL and Qihoo 360 browser.
* GmSSL, not fully functional, can't negotiate CTLS sessions with other hardware/software implementations and it's buggy for the CTLS feature.

## BSC-OpenSSL Details

### Version Scheme

BSC-OpenSSL has its own version scheme which is pretty similar to original OpenSSL's scheme. By executing `openssl version`, you can get the following result:

```
BSC-OpenSSL 7.0.0-beta1
(OpenSSL 1.1.1a-dev  xx XXX xxxx)
```

* the first line indicates the BSC-OpenSSL version
* the second line indicates the base OpenSSL version, in this case it's based on OpenSSL 1.1.1 branch

BSC-OpenSSL numeric release version identifier is described as follows:

* MNNFFSSS: major minor fix status
* The status nibble has one of the values 0 for development, 1 to e for betas 1 to 14, and f for release.
* There is no 'patch' digital or letter

### Additional APIs

For using CTLS with BSC-OpenSSL, several new APIs are added on top of original OpenSSL.

#### `CTLS_method`

#### `CTLS_client_method`

#### `CTLS_server_method`

#### `SSL_is_ctls`

Check if an `SSL` object is performing CTLS protocol.

#### `SSL_CTX_use_enc_PrivateKey`

This function is similar to `SSL_CTX_use_PrivateKey` instead of setting the private key as an encryption private key.

#### `SSL_CTX_use_enc_certificate`

This function is similar to `SSL_CTX_use_certificate` instead of setting the certificate as an encryption certificate.

#### `SSL_CTX_use_enc_PrivateKey_file`

This function is similar to `SSL_CTX_use_PrivateKey_file` instead of setting the private key as an encryption private key.

#### `SSL_CTX_use_enc_certificate_file`

This function is similar to `SSL_CTX_use_certificate_file` instead of setting the certificate as an encryption certificate.

#### `SSL_CTX_use_sign_PrivateKey`

This function is similar to `SSL_CTX_use_PrivateKey` instead of setting the private key as a signature private key.

#### `SSL_CTX_use_sign_certificate`

his function is similar to `SSL_CTX_use_certificate` instead of setting the certificate as a signature certificate.

#### `SSL_CTX_use_sign_PrivateKey_file`

This function is similar to `SSL_CTX_use_PrivateKey_file` instead of setting the private key as a signature private key.

#### `SSL_CTX_use_sign_certificate_file`

This function is similar to `SSL_CTX_use_certificate_file` instead of setting the certificate as a signature certificate.

### How to Test

By using BSC-OpenSSL's `s_client` and `s_server` can test the basic functionality of CTLS.

A bundle of testing SM2 certificates are shipped with the BSC-OpenSSL package. They can be used to do the tests.

Besides you can also test the compatibility against other implementations listed in the [Major Implementations](Major_Implementations) section.

Examples with BSC-OpenSSL's `s_client` and `s_server`.

#### Server Command

```
openssl s_server -accept 1234 -sign_cert /path/to/SS.cert.pem -sign_key /path/to/SS.key.pem -enc_cert /path/to/SE.cert.pem -enc_key /path/to/SE.key.pem -cert /path/to/servercert.pem -key /path/to/serverkey.pem
```

#### CTLS Client with `ECC-SM2-WITH-SM4-SM3`

```
openssl s_client -connect 127.0.0.1:1234 -cipher "ECC-SM2-WITH-SM4-SM3" -ctls
```

#### CTLS Client with `ECDHE-SM2-WITH-SM4-SM3`

```
openssl s_client -connect 127.0.0.1:1234 -cipher "ECDHE-SM2-WITH-SM4-SM3" -sign_cert /path/to/CS.cert.pem -sign_key /path/to/CS.key.pem -enc_cert /path/to/CE.cert.pem -enc_key /path/to/CE.key.pem -ctls
```

#### TLSv1.3 Client

```
openssl s_client -connect 127.0.0.1:1234
```
