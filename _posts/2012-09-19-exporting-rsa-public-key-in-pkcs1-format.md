---
layout: post
title: "Exporting RSA public key in PKCS#1 format"
date: 2012-09-19 20:00:00 +0530
tags: [dotnet, crypto]
---
Recently I was working on task in which I had to generate a RSA key pair and export the public key to another application in PEM format. I implemented the task with help of BouncyCastle(code below) but the exported public key is not accepted by the other application.
```c#
static void ExportPublicKey(AsymmetricKeyParameter publicKey)
{
    var stringWriter = new StringWriter();
    var pemWriter = new PemWriter(stringWriter);
    pemWriter.WriteObject(publicKey);
    stringWriter.Flush();
    stringWriter.Close();

    File.WriteAllText(@"PublicKey.pem", stringWriter.ToString());
}
```
Little investigation revealed that the other application expects the public key in PKCS#1 format whereas the above code exports in PKCS#8 format(the ASN.1 structure of SubjectPublicKeyInfo). The difference between these formats is the additional field algorithm identifier in PKCS#8 format. We can clearly in there ASN.1 structure below

![PKCS vs RSA](/assets/pkcs-public-key-vs-rsa-public-key.png)

Also the exported PEM will have different start/end line like below

in PKCS#8

```
-----BEGIN PUBLIC KEY-----
...
-----END PUBLIC KEY-----
```

but in PKCS#1

```
-----BEGIN RSA PUBLIC KEY-----
...
-----END RSA PUBLIC KEY-----
```

Though the difference is one additional information but exporting public key without that information is not straightforward in BouncyCastle, or atleast at the time of writing. In OpenSSL(libcrypto) this is as simple as the calling the functions `PEM_write_RSAPublicKey` or `PEM_write_bio_RSAPublicKey`. In BouncyCastle the following code exports the public key in PKCS#1 format
```c#
static void ExportRsaPublicKey(AsymmetricKeyParameter publicKey)
{
    var subjectPublicKeyInfo = SubjectPublicKeyInfoFactory.CreateSubjectPublicKeyInfo(publicKey);
    var rsaPublicKeyStructure = RsaPublicKeyStructure.GetInstance(subjectPublicKeyInfo.GetPublicKey());
    var rsaPublicKeyPemBytes = Base64.Encode(rsaPublicKeyStructure.GetEncoded());

    var stringBuilder = new StringBuilder();

    stringBuilder.AppendLine("-----BEGIN RSA PUBLIC KEY-----");

    for (int i = 0; i < rsaPublicKeyPemBytes.Length; ++i)
    {
        stringBuilder.Append((char)rsaPublicKeyPemBytes[i]);

        // wraps after 64 column
        if (((i + 1) % 64) == 0)
            stringBuilder.AppendLine();
    }

    stringBuilder.AppendLine();
    stringBuilder.AppendLine("-----END RSA PUBLIC KEY-----");

    File.WriteAllText(@"RsaPublicKey.pem", stringBuilder.ToString());
}
```

and the code that generates RSA key pair is

```c#
static void Main(string[] args)
{
    var rsaKeyPairGenerator = new RsaKeyPairGenerator();
    rsaKeyPairGenerator.Init(new KeyGenerationParameters(new SecureRandom(), 2048));
    var keyPair = rsaKeyPairGenerator.GenerateKeyPair();

    ExportPublicKey(keyPair.Public);
    ExportRsaPublicKey(keyPair.Public);
}
```