---
layout: post
title: "Comparing X509 distinguished names with BouncyCastle"
date: 2012-11-25 21:55:00 +0530
tags: [dotnet, crypto]
---
In this post I am going to show how to compare X509 distinguished name using BouncyCastle. Distinguished names are used to specify the subject and issuer identity in X509 certificate. Distinguished name(DN) is like a LDAP directory name and contains one or more relative distinguished name(RDN) separated by comma. The definition for distinguished name can be found in X500 specification.

The problem with comparing distinguished name is the relative distinguished names they contain can be in any order. So simple string comparison will not work and parsing the distinguished name is not straight forward.

But BouncyCastle provides a class named `X509Name` through which we can easily parse and compare the X509 distinguished names. The below code snippets demos the parsing and comparison of X509 distinguished names
```c#
static void Main(string[] args)
{
    var name1 = new X509Name("CN=person1, OU=dept1, O=org1");
    var name1Dup = new X509Name("O=org1, OU=dept1, CN=person1");
    var name2 = new X509Name("CN=person2, OU=dept2, O=org1");

    Console.WriteLine("name1 = {0}", name1);
    Console.WriteLine("name2Dup = {0}", name1Dup);
    Console.WriteLine("name2 = {0}", name2);
    Console.WriteLine();

    Console.WriteLine("name1 == name1Dup => {0}", name1.Equivalent(name1Dup));
    Console.WriteLine("name1 == name2 => {0}", name1.Equivalent(name2));
}
```

and the output

```
name1 = CN=person1,OU=dept1,O=org1
name2Dup = O=org1,OU=dept1,CN=person1
name2 = CN=person2,OU=dept2,O=org1

name1 == name1Dup => True
name1 == name2 => False
```
The X509Name class makes it so simple to parse and manipulate the X509 distinguished names. But I encountered a situation where one of the relative distinguished name contained by X509 distinguished name is not recognizable by `X509Name` even though the underlying OID(object identifier) of the relative distinguished name is supported by the `X509Name`. The problematic relative distinguished name is `dnQualifier` whose OID is 2.5.4.46. The same OID is supported and parsable by X509Name if the relative distinguished name is `DN` instead `dnQualifier`.

Digging little deep into X509Name source reveals that we can make it to parse/support `dnQualifier` relative distinguished name like below
```c#
static void Main(string[] args)
{
    X509Name.DefaultLookup["dnqualifier"] = X509Name.DnQualifier;

    var name1 = new X509Name("CN=person1, OU=dept1, O=org1, dnQualifier=aaaabbbbccccddddeeeeffffggg=");
    Console.WriteLine(name1);
}
```
In the above code basically we are providing another name(`dnQualifier`) for `X509Name.DnQualifier` which is by default identified with name `DN`.