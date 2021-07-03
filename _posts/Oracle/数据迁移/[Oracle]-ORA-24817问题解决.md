---
title: [Oracle]-ORA-24817问题解决
date: 202-11-17
---



### MOS参考

Calling OCILobRead and retrieving a CLOB in one piece on 32-bit platforms may fail with: ORA-24817: Unable to allocate the given chunk for current lob operation (Doc ID 780346.1) elevance on 1st Jun 2015***
SYMPTOMS
Reading a CLOB in a single piece using OCILobRead may fail with:

ORA-24817: Unable to allocate the given chunk for current lob operation
when the chunk size or lob piece is greater than 64mb on 32-bit platforms.

#### CAUSE

OCI imposes a limit on the size of the chunks or clob pieces that can be processed by OCILobRead. The max buffer size calculation is based on character set, database version and platform. For 32-bit operating systems, the limit on the buffer size or lob piece is 64MB. It is not reasonable to expect that you can process a CLOB which can be up to 2GB in size as one piece.

#### SOLUTION

Most multi-byte character set store data using a range of 1 - 4 bytes. In order not to exceed the 64MB limitation for a 32-bit OS, choose a chunk size that ensures you will not exceed this limit. i.e 64MB/4 (The maximum buffer size / max bytes needed to store a character for your database characterset).

#### REFERENCES

BUG:3774546 - OCI PROGRAM. OCILOBREAD() CRASH IN KGHALO AFTE 11 MB CLOB SIZE.

