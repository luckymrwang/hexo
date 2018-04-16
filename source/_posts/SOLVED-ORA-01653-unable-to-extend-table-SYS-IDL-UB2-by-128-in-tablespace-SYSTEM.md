title: >-
  [SOLVED] ORA-01653: unable to extend table SYS.IDL_UB2$ by 128 in tablespace
  SYSTEM
date: 2018-03-23 18:32:05
tags: [Oracle]
---

### Problem

```
Error report:
ORA-00604: error occurred at recursive SQL level 1
ORA-01653: unable to extend table SYS.IDL_UB2$ by 128 in tablespace SYSTEM
00604. 00000 -  "error occurred at recursive SQL level %s"
*Cause:    An error occurred while processing a recursive SQL statement
(a statement applying to internal dictionary tables).
*Action:   If the situation described in the next error on the stack
can be corrected, do so; otherwise contact Oracle Support.
```

<!-- more -->
### Validation

```
SELECT TABLESPACE_NAME
, FILE_NAME
, BYTES / 1024 / 1024 AS BYTES_MB
, AUTOEXTENSIBLE
, MAXBYTES  / 1024 / 1024 AS MAXBYTES_MB
FROM DBA_DATA_FILES;
```

### Solution

```
ALTER DATABASE DATAFILE 'D:\ORACLEXE\APP\ORACLE\ORADATA\XE\SYSTEM.DBF'
AUTOEXTEND ON NEXT 1M MAXSIZE 1024M;
```
