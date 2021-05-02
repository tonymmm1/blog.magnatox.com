---
title: "C File IO"
date: 2020-07-23T19:50:09-05:00
draft: false
tags: [
    "C",
    "Programming",
]
---

C file io tutorial with examples. 

```c
#include <stdio.h>

int main()
{
    FILE *fp;                   /* file pointer */
    char *filepath = "test.txt" /* file location */

    fopen(filepath, "r");       /* fopen("string", "mode") */

    /* perform some operation */

    fclose(fp);                 /* closes file */
    return 0;                   /* return code 0 indicates program successfully ran */
}
```

In the example above the program sets a file pointer variable to use to store file data. Next it create a char pointer to store the file path to be used. Then it performs the file operation in read mode.

When working with files it is recommended to set some sort of an error check to ensure that file is actually accessible.

```c
#include <stdio.h>  /* standard C library */
#include <stdlib.h> /* adds exit() functionality */

int main()
{       
    FILE *fp;                       /* file pointer */
    char *file_path = "test.txt";   /* file location */

    fopen(file_path, "r");          /* fopen("string", "mode") */
    if (fp == NULL)                 /* if file is NULL */
    {
        printf("unable to open file %s\n", file_path);  /* error output */
        exit(1);    /* exits program */
    }

    /* perform some operation */

    fclose(fp); /* closes file */
    return 0; /* return code 0 indicates program successfully ran */
}
```

Many other file modes can be found in the references section below. 

[C Programming](/posts/c_programming)

## References:

* [modes](https://pubs.opengroup.org/onlinepubs/009695399/functions/fopen.html)
