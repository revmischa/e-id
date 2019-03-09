# What is this?
Instructions for interacting with an identity smartcard.

The examples here are using an Estonian e-Residency identity.

Shows how to sign files, verify identities and signatures, and a look at what's in your card and identity.



## Use Hardware

### Packages:
_Ubuntu_: sudo apt install opensc
To install support apps for signing documents and chrome extension integration on debian/ubuntu:  https://id.ee/index.php?id=34448

#### List attached readers:
```
$ opensc-tool -l
# Detected readers (pcsc)
Nr.  Card  Features  Name
0    Yes             ACS ACR 38U-CCID 00 00
```
#### List keys:
```
$ pkcs15-tool -k
Using reader with a card: ACS ACR 38U-CCID 00 00
Private EC Key [Isikutuvastus]
        Object Flags   : [0x1], private
        Usage          : [0x104], sign, derive
        Access Flags   : [0x1D], sensitive, alwaysSensitive, neverExtract, local
        FieldLength    : 384
        Key ref        : 1 (0x1)
        Native         : yes
        Auth ID        : 01
        ID             : 01
        MD:guid        : c6a6b626-f80e-3aae-e50f-5fc305a5ff09

Private EC Key [Allkirjastamine]
        Object Flags   : [0x1], private
        Usage          : [0x200], nonRepudiation
        Access Flags   : [0x1D], sensitive, alwaysSensitive, neverExtract, local
        FieldLength    : 384
        Key ref        : 2 (0x2)
        Native         : yes
        Auth ID        : 02
        ID             : 02
        MD:guid        : 641de276-8ed4-8882-f8c4-c517647136ad
```
#### List PINs:
```
$ pkcs15-tool --list-pins
Using reader with a card: ACS ACR 38U-CCID 00 00
PIN [PIN1]
        Object Flags   : [0x0]
        Auth ID        : 03
        ID             : 01
        Flags          : [0x00]
        Length         : min_len:4, max_len:12, stored_len:12
        Pad char       : 0x00
        Reference      : 1 (0x01)
        Type           : ascii-numeric
        Tries left     : 2

PIN [PIN2]
        Object Flags   : [0x0]
        Auth ID        : 03
        ID             : 02
        Flags          : [0x00]
        Length         : min_len:5, max_len:12, stored_len:12
        Pad char       : 0x00
        Reference      : 2 (0x02)
        Type           : ascii-numeric
        Tries left     : 3

PIN [PUK]
        Object Flags   : [0x40]
        ID             : 03
        Flags          : [0x40], unblockingPin
        Length         : min_len:8, max_len:12, stored_len:12
        Pad char       : 0x00
        Reference      : 0 (0x00)
        Type           : ascii-numeric
        Tries left     : 3
```

## Sign a file

1. Make a hash of your file.
  `wget http://mischa.lol/eeid/omgwall.txt`
  `openssl dgst -binary -sha512 omgwall.txt > omghash`

2. Then sign the hash of your file:

  `pkcs15-crypt --sign --key 02 --sha-512 --raw -i omghash -f openssl > omgwall.openssl.sig`
  Generates a binary openssl signature file signed by your key on the card.
  Key 01 is for authentication, key 02 is for signing.


## Verify a signature

### Get signer certificate:

#### macOS:
Export your signing certificate from DigiDoc 4 client to Apple Keychain, and then export it from Keychain to a PEM file.

#### Linux:
You can export your public key from the card with:
`pkcs15-tool --read-public-key 2 > eeSigningCert.pem`

### Get public key:
`openssl x509 -pubkey -noout -in eeSigningCert.pem > publicKey.pem`

You can then use openssl to verify the signature, proving the owner of the certificate signed the file:
```
$ openssl dgst -verify publicKey.pem -signature omgwall.openssl.sig -sha512 omgwall.txt
Verified OK
```

## Verify the identity of the signer
Who signed this file? Is their public key signed by the Estonian Police and Border Guard?

* Get a root CA and intermediate.
  * Estonia root CA: https://www.sk.ee/upload/files/EE_Certification_Centre_Root_CA.pem.crt
  * Intermediate: https://www.sk.ee/upload/files/ESTEID-SK_2015.pem.crt
* Verify certificate chain with eeSigningCert.pem:
```
$ openssl verify -show_chain -CAfile EE_Certification_Centre_Root_CA.pem.crt -untrusted ESTEID-SK_2015.pem.crt eeSigningCert.pem
eeSigningCert.pem: OK
Chain:
depth=0: C = EE, O = ESTEID (DIGI-ID E-RESIDENT), OU = digital signature, CN = "SPIEGELMOCK,MISCHA,38511060182", SN = SPIEGELMOCK, GN = MISCHA, serialNumber = 38610030182 (untrusted)
depth=1: C = EE, O = AS Sertifitseerimiskeskus, organizationIdentifier = NTREE-10747013, CN = ESTEID-SK 2015 (untrusted)
depth=2: C = EE, O = AS Sertifitseerimiskeskus, CN = EE Certification Centre Root CA, emailAddress = pki@sk.ee
```
