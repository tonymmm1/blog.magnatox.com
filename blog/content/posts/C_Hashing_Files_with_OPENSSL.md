---
title: "C Hashing Files With OpenSSL"
date: 2020-07-23T20:49:40-05:00
draft: false
tags: [
	"C",
	"Programming",
]
---

Hashing files with C is actually quite easily accomplished using existing [OpenSSL](https://www.openssl.org/) library headers. OpenSSL is quite commonly used for many applications such as for TLS/SSL based web servers or even basic cryptographic tasks such as file hashing and encryption. 

# Requirements:

* OpenSSL headers

Make sure to compile the following programs using the -lcrypto flag.

```
gcc -o main main.c -lcrypto
```

```
/* String hashing example */
#include <stdio.h>
#include <string.h>
#include <openssl/evp.h>	/* OpenSSL EVP headers for hashing */

#define BUFF_SIZE 4096	/* buffer size */

int main()
{
	size_t i;					/* loop variables */
	char buff[BUFF_SIZE];				/* buffer for hash */
	unsigned int md_len;				/* stores hash length */
	EVP_MD_CTX *mdctx;				/* stores hash context */
	unsigned char md_value[EVP_MAX_MD_SIZE];	/* stores hash */
	char *input = "test hash string";		/* input */

	mdctx = EVP_MD_CTX_new();	/* Initialize new ctx */

	const EVP_MD *EVP_md5();			/* use EVP md5 function */

	EVP_DigestInit_ex(mdctx, EVP_md5(), NULL);	/* Initializes digest type */
	
	EVP_DigestUpdate(mdctx, input, strlen(input));	/* Add input to hash context */
	
	EVP_DigestFinal_ex(mdctx, md_value, &md_len);	/* Finalize hash context */
	
	for (i = 0; i < md_len; i++)	/* loops through hash length */
	{
		printf("%02x", md_value[i]);	/* prints 2 hex-values of hash per loop */
	}
	printf("  %s\n", input);	/* prints out input after hash */
	
	return 0;	/* returns 0 */
}
```

This program example takes in a simple string and hashes it and then displays the hash encoded as hex next to input string. 

```
/* File hashing example */
#include <stdio.h>
#include <stdlib.h>
#include <openssl/evp.h>        /* OpenSSL EVP headers for hashing */

#define BUFF_SIZE 4096  /* buffer size */

int main()
{
        size_t i, n;                                    /* loop variables */
	FILE *fp;					/* file pointer */
        char buff[BUFF_SIZE];                           /* buffer for hash */
        unsigned int md_len;                            /* stores hash length */
        EVP_MD_CTX *mdctx;                              /* stores hash context */
        unsigned char md_value[EVP_MAX_MD_SIZE];        /* stores hash */
        char *input = "/bin/echo";         		/* input file */

	fp = fopen(input, "rb");	/* reads in file */
	if (fp == NULL)
	{
		printf("failed to open file %s", input);
		exit(1);
	}

        mdctx = EVP_MD_CTX_new();       /* Initialize new ctx */

        const EVP_MD *EVP_md5();                        /* use EVP md5 function */

        EVP_DigestInit_ex(mdctx, EVP_md5(), NULL);      /* Initializes digest type */

        while ((n = fread(buff, 1, sizeof(buff), fp)))       /* reads in values from buffer containing file pointer */
        {
                EVP_DigestUpdate(mdctx, buff, n);       /* Add buffer to hash context */
        }

        EVP_DigestFinal_ex(mdctx, md_value, &md_len);   /* Finalize hash context */

        for (i = 0; i < md_len; i++)    /* loops through hash length */
        {
                printf("%02x", md_value[i]);    /* prints 2 hex-values of hash per loop */
        }
        printf("  %s\n", input);        /* prints out input after hash */

        return 0;       /* returns 0 */
}
```

The program above takes in a file path containing a file and then diplays the hash encoded as a hex string next to the input file.

## References:

* [OpenSSL](https://www.openssl.org/)
