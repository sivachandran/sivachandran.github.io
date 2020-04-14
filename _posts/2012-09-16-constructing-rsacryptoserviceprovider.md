---
layout: post
title: "Constructing .NET RSACryptoServiceProvider from DER bytes"
date: 2012-09-16 20:30:00 +0530
tags: [dotnet, crypto]
---
In my current task I have to construct private key object from its DER bytes. In OpenSSL I can easily do this with d2i_RSAPrivateKey function but in .NET this doesn't seems to be an easy task. I hoped in [BouncyCastle](http://www.bouncycastle.org/csharp/) this can be done in single method call but it requires the following code to achieve the functionality
```c#
byte[] privateKeyDer = File.ReadAllBytes("PrivateKey.der");
var derSequence = new Asn1InputStream(privateKeyDer).ReadObject();
var privateKeyStructure = new RsaPrivateKeyStructure((Asn1Sequence)derSequence);

var privateCrtKeyParameters = 
    new RsaPrivateCrtKeyParameters(privateKeyStructure.Modulus,
                                   privateKeyStructure.PublicExponent,
                                   privateKeyStructure.PrivateExponent,
                                   privateKeyStructure.Prime1,
                                   privateKeyStructure.Prime2,
                                   privateKeyStructure.Exponent1,
                                   privateKeyStructure.Exponent2,
                                   privateKeyStructure.Coefficient);

var privateKey = DotNetUtilities.ToRSA(privateCrtKeyParameters); // returns RSA object
```

Please note the classes `Asn1InputStream`, `RsaPrivateKeyStructure`, `RsaPrivateCrtKeyParameters` and `DotNetUtilities` are provided by BouncyCastle.

***03-Nov-2013 Update***
I recently realized that it is possible for the RSA components(e.g. modulus) to have lesser bits than the actual RSA key's bit size. For example, in one case the modulus is 255 bytes(2040 bits) instead of expected 256 bytes for a 2048 bits RSA key. Some validation code inside `RSACryptoServiceProvider.ImportParameters` method rejects these kind of RSA key parameters with exception `Invalid Parameter` as the bytes size is lesser than expected. I don't think it is a problem with BouncyCastle's BigInteger implementation as OpenSSL also encodes these big integers with fewer bytes like the BouncyCastle. But OpenSSL accepts these big integers without any issues but it is only the `RSACryptoServicer.ImportParameters` method that rejects it.

We can make the `ImportParameters` to work with these kind of big integers by padding zeros to the big integers. You can find the implementation below, so instead of calling `DotNetUtilities.ToRSA()` we should call our method `ToRsa()` to convert the RSA key parameters to RSA object.
```c#
private static byte[] PadLeft(byte[] bytes, int sizeToPad)
{
    if (bytes.Length &lt; sizeToPad)
    {
        // byte default value is zero, so the paddedBytes array is filled with zero by default
        var paddedBytes = new byte[sizeToPad];

        Buffer.BlockCopy(bytes, 0, paddedBytes, sizeToPad - bytes.Length, bytes.Length);
        bytes = paddedBytes;
    }
    
    return bytes;
}

public static System.Security.Cryptography.RSA ToRsa(RsaPrivateCrtKeyParameters privateKey, int bitSize)
{
    var rsaParameters = new System.Security.Cryptography.RSAParameters();
    rsaParameters.Modulus = PadLeft(privateKey.Modulus.ToByteArrayUnsigned(), bitSize / 8);
    rsaParameters.Exponent = PadLeft(privateKey.PublicExponent.ToByteArrayUnsigned(), 3);
    rsaParameters.D = PadLeft(privateKey.Exponent.ToByteArrayUnsigned(), bitSize / 8);
    rsaParameters.P = PadLeft(privateKey.P.ToByteArrayUnsigned(), bitSize / 16);
    rsaParameters.Q = PadLeft(privateKey.Q.ToByteArrayUnsigned(), bitSize / 16);
    rsaParameters.DP = PadLeft(privateKey.DP.ToByteArrayUnsigned(), bitSize / 16);
    rsaParameters.DQ = PadLeft(privateKey.DQ.ToByteArrayUnsigned(), bitSize / 16);
    rsaParameters.InverseQ = PadLeft(privateKey.QInv.ToByteArrayUnsigned(), bitSize / 16);

    var rsa = new System.Security.Cryptography.RSACryptoServiceProvider();
    rsa.ImportParameters(rsaParameters);

    return rsa;
}
```