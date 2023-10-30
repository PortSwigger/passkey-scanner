# PasskeyScanner
This is a BurpSuite plugin that recognizes and scans Passkey (webauthn) protocols and detects security issues. 
## Verifying the public key algorithms and elliptic curves
When a key pair is created for credentials, the Relaying Party and Authenticator need to agree on an acceptable algorithm. This process starts with the Relaying Party sending list of acceptable algorithms in order of preference. The algorithms are encoded using the IANA CBOR Object Signing and Encryption (COSE) algorithm registry (https://www.iana.org/assignments/cose/cose.xhtml#algorithms). PasskeyScanner will scan this list and report unacceptable algorithms. The registry includes a “recommended’ field, so we use this guidance as a starting point to determine which algorithms are acceptable or not. This approach is not perfect for two cases. 
1)	Many of the algorithms are symmetric encryption algorithms, which are not appropriate for Passkeys. Symmetric algorithms are reported as a finding if they’re found in the list of supported algorithms.
2)	RSA PKCS v1.5 is a signature scheme based on RSA public keys that is still commonly used. The PKCS v1.5 structure has led to padding oracle attacks in several implementations of this format (CVE-2022-4304, CVE-2023-0361, CVE-2023-4421, CVE-2020-25659, CVE-2020-25657, etc.). RFC 3447 (2003) PKCS #1 RSA Cryptography Specification (https://www.rfc-editor.org/rfc/rfc3447#section-8) recommends that PKCS v1.5 is not adopted for new applications, so we follow this advice.
If the authenticator decides to generate an elliptic curve keypair for a credential, the public key is returned to the Relaying Party. The public key is CBOR encoded within the “attestationObject”. This plugin will decode the “attestationObject” and verify that the encoded algorithm identifier matches with the curve specified in the public key (i.e. ES256 algorithm has a public key on curve SECP256R1).  If an algorithm and public key curve mismatch is found a finding will be reported.
## User verification
Passkeys provide a feature to enforce user verification during authentication. For users using a mobile device this is typically a biometric verification. If user verification is not “required” we report a finding since this feature provides additional security in the case that an authenticator is lost or stolen. Windows will often perform this verification by requiring the user to enter the login credentials, which may become a usability constraint on the user.  If the user verification is set to “preferred” this will allow the passkey implementation to determine if user verification is required, which may be an appropriate balance of usability and risk for some applications.
## Strong authentication challenge
To demonstrate proof of the secret key WebAuthN requires the authenticator to cryptographically sign an authentication challenge. If the challenge is predictable or an attacker can bias the challenge string, then the cryptographic strength of the authentication ceremony is weakened. WebAuthN requires that the challenge is at least 16 bytes (128 bits) and a length of 32 bytes (256 bits) is commonly implemented. If a challenge is less that 16 bytes, a finding is reported.
## “AllowCredentials” information disclosure
The “AllowCredentials” parameter is an optional list that can be provided to the authentication operation. It specifies a list of credential IDs that is acceptable to the Relaying Party. It is intended to support “non-resident” or “server-side” credentials, which maintains backward compatibility older WebAuthN authenticators. The “AllowCredentials” array is a list of credential objects which contain an id and type field.  With Passkeys the credential type will always be “public-key” but a system may also allow other credential types. If the RelayingParty provides a listed of allowed credentials that includes a weaker form of authentication such as “password” this would provide an attacker with useful information of which accounts to target in a password attack. If any credential type other than “public-key” is provided a finding is reported.
## User Handle Personally Identifiable Information
The user handle is the “id” field of a user object that may be stored with the credential by the authenticator. To protect user privacy a user identifier should not be personally identifiable information. An authenticator may reveal user handle without verification of an authentication operation from the user, which could allow a script to perform user enumeration. This also prevents user enumeration attacks against a Relaying Party where a user attempts to enumerate users of the site by attempting to authenticate many user accounts.
