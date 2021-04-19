---
title:PostgreSQL-DBA常用命令
date:2020-07-04
---



##### PostgreSQL-DBA常用命令

- [基本命令](基本命令)
- [内置函数](内置函数)
- [源码学习资源](源码学习资源)
- [I/O固定扇区](I/O固定扇区)



### 基本命令

查看当前最小版本号

```
show server_version_num;
--启动数据库集簇 
pg_ctl -D $PGDATA -l logfile start 

--创建新的数据库 
createdb test 

--创建表空间
create tablespace new_tblspc location '/home/postgres/tblspc';

--查看表空间名称及oid
select oid,spcname from pg_tablespace；
```



### 内置函数

```sql
--OID 与对应的数据文件之间的关系

SELECT relname, oid, relfilenode FROM pg_class WHERE relname = 'sampletbl'; 


--内置函数pg_relation_file path查看表的文件路径
SELECT pg_relation_filepath('sampletbl');

--centos7版本查看进程结构
pstree -p 9687
```

hexdump命令查看postgresql数据文件

```
hexdump -e '16/1 "%02X " "  |  "' -e '16/1 "%_p" "\n"' 49155 > 49155.txt
```

[有助于理解hexdump命令](https://www.bookstack.cn/read/linux-command-1.5.1/command-hexdump.md)



### 源码学习资源

```
http://blog.chinaunix.net/uid-24774106-id-3764606.html
```



### I/O固定扇区

```
linux读取硬盘指定扇区


/* by hugang at 6/8 9:40 */
/* mod by jevan at 31/8 11:56 */
/* read some disk a sector */
#include <stdio.h>
#include <errno.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>

#include <linux/fs.h>

int
get_disk_sector (int fd)
{
  int sectorsize;

  ioctl (fd, BLKSSZGET, &sectorsize);

  return sectorsize;
}

/** 
* read_disk_sector: 
* @dev: raw disk FILE * 
* @sector: 
* return is the disk sectorsize 
* */
extern int errno;
int
read_disk_sector (int fd, unsigned long sector, char **p)
{
  int sectorsize;
  FILE *fp;

/* get disk sector size */
  sectorsize = get_disk_sector (fd);
  if (sectorsize == 0)
    {
      fprintf (stderr, "get disk sector size failed\n");
      return (-1);
    }

/* seek it */
  lseek (fd, 0, SEEK_SET);
  if (lseek (fd, (sectorsize * sector), SEEK_CUR) == -1)
    {
      fprintf (stderr, "seek to %d failed\n", sectorsize * sector);
      return (-1);
    }
/* read it */
  *p = (char *) malloc (sectorsize);
  if (*p == NULL)
    {
      fprintf (stderr, "malloc memory failed\n");
      return (-1);
    }

  return read (fd, *p, sectorsize);
}

/* dump data for display */
void
dump_disk_sector (char *p, int size)
{
  int i;
  for (i = 0; i < size; i++)
    {
      if (i % 16 == 0 && i != 0)
    printf ("\n");
      printf ("%2x ",/* (unsigned int) p & 0xff*/(unsigned char)*p++);
    }

  printf ("\n");
  return;
}

int
main (int argc,char* argv[])
{
  char *d;
  int size;
  int fd;
  char *dev = "/dev/sda";

/* open it */
  fd = open (dev, O_RDONLY);
  if (fd == -1)
    {
      fprintf (stderr, "open %s failed %s\n", dev, strerror (errno));
      return (-1);
    }

  size = read_disk_sector (fd, atoi(argv[1]), &d);

  close (fd);

  if (size <= 0)
    return (0);

  dump_disk_sector (d, size);

  free (d);

  return (0);
}
```