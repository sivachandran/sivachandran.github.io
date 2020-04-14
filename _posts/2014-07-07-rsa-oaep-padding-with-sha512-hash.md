---
layout: post
title: "RSA OAEP padding with SHA512 hash algorithm"
date: 2014-07-07 22:17:00 +0530
tags: [crypto, c-cpp]
---
Recently I wanted to encrypt a message with RSA with OAEP padding. I also wanted to use SHA512 as hashing algorithm and mask generation function(MGF) in the OAEP padding instead of SHA1. But it looks like it is not possible with the OpenSSL/libcrypto as the SHA1 hash algorithm is hard coded in OAEP padding implementation. This is confirmed by this thread in OpenSSL forum. Though the forum thread was written around 2012 but still I couldn't find a way use either SHA256 or SHA512 as my hashing algorithm and MGF in OAEP padding.

As suggested by "Dr Stephen N. Henson"(the core developer of OpenSSL) in the forum thread , I've took the implementation of RSA OAEP padding and modified to use SHA512 instead of SHA1. It is mostly just find `EVP_sha1` and replace with `EVP_sha512`. We also need to update the usage of `SHA_DIGEST_LENGTH` macro to `SHA512_DIGEST_LENGTH` to reflect the output length of SHA512 hash. Below is the modified RSA OAEP padding implementation which uses SHA512 algorithm. Hope it helps, cheers.

```c
static int MGF1_SHA512(unsigned char *mask, long len, const unsigned char *seed, long seedlen)
{
    return PKCS1_MGF1(mask, len, seed, seedlen, EVP_sha512());
}

static int RSA_padding_add_PKCS1_OAEP_SHA512(unsigned char *to, int tlen,
                                             const unsigned char *from, int flen,
                                             const unsigned char *param, int plen)
{
    int i, emlen = tlen - 1;
    unsigned char *db, *seed;
    unsigned char *dbmask, seedmask[SHA512_DIGEST_LENGTH];

    if (flen > emlen - 2 * SHA512_DIGEST_LENGTH - 1)
    {
        RSAerr(RSA_F_RSA_PADDING_ADD_PKCS1_OAEP,
            RSA_R_DATA_TOO_LARGE_FOR_KEY_SIZE);
        return 0;
    }

    if (emlen < 2 * SHA512_DIGEST_LENGTH + 1)
    {
        RSAerr(RSA_F_RSA_PADDING_ADD_PKCS1_OAEP, RSA_R_KEY_SIZE_TOO_SMALL);
        return 0;
    }

    to[0] = 0;
    seed = to + 1;
    db = to + SHA512_DIGEST_LENGTH + 1;

    if (!EVP_Digest((void *)param, plen, db, NULL, EVP_sha512(), NULL))
        return 0;
    memset(db + SHA512_DIGEST_LENGTH, 0,
        emlen - flen - 2 * SHA512_DIGEST_LENGTH - 1);
    db[emlen - flen - SHA512_DIGEST_LENGTH - 1] = 0x01;
    memcpy(db + emlen - flen - SHA512_DIGEST_LENGTH, from, (unsigned int) flen);
    if (RAND_bytes(seed, SHA512_DIGEST_LENGTH) <= 0)
        return 0;

    dbmask = (unsigned char*)OPENSSL_malloc(emlen - SHA512_DIGEST_LENGTH);
    if (dbmask == NULL)
    {
        RSAerr(RSA_F_RSA_PADDING_ADD_PKCS1_OAEP, ERR_R_MALLOC_FAILURE);
        return 0;
    }

    if (MGF1_SHA512(dbmask, emlen - SHA512_DIGEST_LENGTH, seed, SHA512_DIGEST_LENGTH) < 0)
        return 0;
    for (i = 0; i < emlen - SHA512_DIGEST_LENGTH; i++)
        db[i] ^= dbmask[i];

    if (MGF1_SHA512(seedmask, SHA512_DIGEST_LENGTH, db, emlen - SHA512_DIGEST_LENGTH) < 0)
        return 0;
    for (i = 0; i < SHA512_DIGEST_LENGTH; i++)
        seed[i] ^= seedmask[i];

    OPENSSL_free(dbmask);
    return 1;
}
```