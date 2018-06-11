# Welcome to the ACVP wiki!

The goal for this project is to provide a messaging protocol that can be used with existing authentication and communication protocols to enable the automation of testing and validation of NIST-approved cryptographic algorithms.

We are working actively on this exciting project and people are starting to [notice]( http://www.theregister.co.uk/2016/08/01/cisco_nist_want_to_help_devs_escape_the_vulnvalidation_trap/). Thanks!

# Accessing the demo server

See these [detailed instructions](https://github.com/usnistgov/ACVP) first for obtaining TLS credentials.

## Second-Factor Authentication and Authorization Schema for Accessing and Working with the NIST Automated Cryptographic Validation Services 

The second-factor authentication is based on the Time-based One-Time-Password (TOTP) from [RFC 6238](https://tools.ietf.org/html/rfc6238).

### High-Level Architecture. 
The high-level architecture of an enterprise client architecture is shown in the figure below. ![High-Level Architecture](https://github.com/usnistgov/ACVP/blob/master/Images/AuthArchitecture-proxy-server.svg). 

The architecture includes two application servers (ACVP Proxy and TOTP). The application servers shall be deployed behind a firewall. The TOTP server shall be accessible internally only with respect to the enterprise firewall. The primary function of the TOTP server is to house the client TOTP key K (see  [RFC 6238](https://tools.ietf.org/html/rfc6238)) assigned to the person in the enterprise responsible for accessing ACVTS and compute the current OTP upon requests from the ACVP Proxy for completing the two-factor authentication process.  

An alternative high-level architecture of an enterprise client architecture is shown below. ![High-Level Architecture](https://github.com/usnistgov/ACVP/blob/master/Images/AuthArchitecture-client-app.svg). 


The architecture includes one application server (TOTP server), one client machine with the client ACVP application. The TOTP server and client machine shall be deployed behind a firewall. The TOTP server shall be accessible internally only with respect to the enterprise firewall. The primary function of the TOTP server is to house the client TOTP key K (see [RFC 6238](https://tools.ietf.org/html/rfc6238)) assigned to the enterprise for accessing ACVTS and compute the current OTP upon requests from the ACVP Proxy for completing the two-factor authentication process.   

Each participating partner to ACVTS that implements this architecture shall obtain a Base64-encoded client TOTP key upon completing a successful National Voluntary Laboratory Accreditation Program (NVLAP) accreditation. 

### TOTP Parameters

While the TOTP authentication is described in  [RFC 6238](https://tools.ietf.org/html/rfc6238) and  [RFC 4226](https://tools.ietf.org/html/rfc4226), there are several parameters that are important to specify to allow clients to correctly authenticate. 
* The T0 (time) offset from UNIX time shall be zero. 
* Synchronization: the server shall use the Network Time Protocol (NTP) to synchronize with the NIST time server at time.nist.gov. It is expected that clients use a corresponding methodology to prevent clock skew. 
* The time step (or X as specified in RFC 6238) shall be 30 seconds. 
* Cryptographic primitives: HMAC-SHA-256 based on SHA-256 shall be used in this implementation, a valid option per RFC 6238. Please refer to Appendices A and B in RFC 6238 for details on how to compute the TOTP value with these cryptographic primitives and validate the correctness of your client implementation.  
* TOTP length: the number of digits produced based on RFC 4226 shall be 8 digits. Note that any leading zeros in the TOTP value shall be preserved up to the 8-digit length.

As part of the authorization process there is the option on the part of the server to allow a “window,” checking OTP values a specified number of steps either forward or backward from the current step, to handle both network delays and possible clock skew. RFC 6238 recommends checking one step backward to accommodate any network delay and provides no specific guidance on a window for a clock skew other than to recommend this window remain small. There will be a small window, likely one step backward and forward, but this value may change over time based on security and operational considerations.

### Two-factor authentication flows
The ACVTS server has a dedicated RESTful entry point for client authentication: https://demo.acvts.nist.gov/acvp/validation/acvp/login. The corresponding two-factor authentication flow is shown in the figure below. Note that the successful first-factor authentication is a prerequisite for performing the second factor authentication with TOTP. Note also that this workflow utilizes two protocols: Transmission Control Protocol and Internet Protocol (TCP/IP) and Hypertext Transfer Protocol Secure (HTTPS). ![Authentication flows](https://github.com/usnistgov/ACVP/blob/master/Images/2-factor-auth-proxy-server.svg). 

The NIST ACVTS server links a client ID to a successful first-factor authentication upon completing the two-way TLS handshake for the just established session with the client ID corresponding to the client certificate used in the handshake. The NIST ACVTS server and the NIST TOTP server maintain a database with client ID’s which must be synchronized for proper operation of the second-factor authentication. The NIST TOTP server uses the client ID to find the shared secret K corresponding to that ID. The NIST TOTP server then computes an OTP using the algorithm from  [RFC 6238](https://tools.ietf.org/html/rfc6238). The NIST TOTP server next compares the computed OTP with the supplied client OTP value. If the two match, within a server defined window, the client is authenticated, otherwise the authentication attempt is rejected. Correspondingly, the ACVTS server creates an Authorization JSON Web Token (JWT) and returns it to the client upon success.

### Authorization flows

Successful client authentication is required for performing any work with the ACVTS server. In addition, performing specific validation actions or queries on ACVTS requires corresponding authorization. This section defines the basic workflows with the corresponding authorization required for them. A fundamental assumption for this definition is that once the first factor authentication is established with the two-way TLS, ACVTS will manage the workflow sessions for clients using JWT, separately from the session management in TLS. This section considers only the workflow session management with JWT with the assumption that the underlying TLS session between a given client and the ACVTS server is independently maintained.

The workflow authorization flows are shown in the figure below. ![Authorization flows](https://github.com/usnistgov/ACVP/blob/master/Images/authorization-flows.png) 

Note that all HTTP requests listed are over TLS. Note also that to re-authenticate in this context when the JWT is expired the client needs to submit only the new OTP value using the TLS session previously established. In other words, the client would need to perform only the actions shown in Figure 4 that correspond to the second factor authentication and will reuse the 2-way TLS session as a first factor authentication as long as the TLS session is active.   
The basic process will flow as follows. 

1. POST the following to **/validation/acvp/login** <br/>
```
[
   {"acvVersion": "0.x"},
   {"password":"<ONE TIME PASSWORD>"}
]
```
Note: That the “password” value is a string and the one-time password could include one or more leading zeros that are expected to be provided up to the number of digits specified above.

2. Receive the response  
```
[
   {"acvVersion": "0.x"},
   {"accessToken": "<ACCESS TOKEN>"}
]
```
3. POST the registration to **/validation/acvp/register** with the following in the header: 
Authorization: Bearer `<accessToken>`

4.  Receive the response
```
[
   {"acvVersion": "0.4"},
   {"capabilityResponse": {"vectorSets": [{"vsId": <VECTOR SET ID>}]},
    "testSession": {"testId": <TEST SESSION ID>},
    "accessToken": "<ACCESS TOKEN>"}
]
```
NOTE: The access token will be different than one the received from **/validation/acvp/login**, and should be used for all the subsequent requests for this test session. 

5.  GET **/validation/acvp/vectors?vsId=`<VECTOR SET ID>`** with following in the header
Authorization: Bearer `<accessToken>`

6. Receive the vector set 

7. POST the answers to **/validation/acvp/vectors?vsId=`<VECTOR SET ID>`** with following in the header
Authorization: Bearer `<accessToken>`

8. GET **/validation/acvp/results?vsId=`<VECTOR SET ID>`** with following in the header
Authorization: Bearer `<accessToken>`

9. Receive the results for the vector set

The access tokens received from either the **/validation/acvp/login** endpoint or the **/validation/acvp/register** endpoint are set to expire after a pre-defined period, and will result in a “401 Unauthorized” HTTP status code when used outside of this window. The following process will allow the client to refresh the token. 

1. POST the following to **/validation/acvp/login**
```
[
   {"acvVersion": "0.x"},
   {"password":"<ONE TIME PASSWORD>",
    "accessToken":"<EXPIRED ACCESS TOKEN>"}
]
```
2. Receive the response
```
[
   {"acvVersion":"0.x"},
   {"accessToken":"<ACCESS TOKEN>"}
]
```

This access token can now be used to continue working with the test session as described in the previous workflow.

### Additional considerations
The TLS protocols used for authentication shall be version 1.2 or higher. The ciphersuites chosen for each TLS session shall use Federal Information Processing Standard (FIPS)-approved and validated algorithm primitives. The negotiated ciphersuite for each TLS session shall use an authenticated encryption with additional data (AEAD) symmetric block cipher, such as the Galois Counter mode of the Advanced Encryption Standard [AES-GCM](https://doi.org/10.6028/NIST.SP.800-38D).  

The Partner ACVP Proxy server and the Partner TOTP server shall be deployed on the enterprise intranet and may use other protocols, such as the Internet Protocol Security (IPSec) for communication between themselves. If the Internet Key Exchange version 2 (IKEv2) protocol is used for key establishment, it shall use FIPS-approved and validated algorithm primitives. The negotiated ciphersuite for the resulting session shall use an AEAD symmetric block cipher, such as [AES-GCM](https://doi.org/10.6028/NIST.SP.800-38D).
 
The TOTP server shall authenticate the Proxy server or the ACVP application before sending any authentication data. In the case of IPSec with a pre-shared key, the authentication of the Proxy server is implicit.  

### Provisioning of shared secrets for TOTP

To deploy and use TOTP in practice one must provision the shared secret K on the client TOTP server and the NIST TOTP server. We refer to Section 7.5 in [RFC 4226](https://tools.ietf.org/html/rfc4226) for best practices around generation and management of shared secrets for OTP. 



