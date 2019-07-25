---
title: 使用bash命令清理MSSQL内存
desc: 刷下博客活跃度
author: ngtmuzi
category: 神秘代码
date: 2019-07-03 16:46:59
tags: 
- Bash
- MSSQL
---

首先你得在windows环境上有一个bash环境，比如 [git-bash](https://gitforwindows.org) 或 [cygwin64](https://cygwin.com/index.html)

加了一个是否有活跃sql会话的判断，避免搞出问题，sql语句部分来源于热心网友

```bash
#释放mssql的内存
. /etc/profile

sessionNum=`osql -E -S . -Q "select count(1) from sys.dm_exec_sessions where status!='sleeping'" | sed -n 3p | tr -d [:blank:] | tr -d [:cntrl:]`

if [ `expr $sessionNum` -ne 1 ]; then
	echo "存在正在执行的sql server会话，取消释放内存";
	exit 0;
fi

osql -E -S . -Q "
DBCC FREEPROCCACHE
DBCC FREESESSIONCACHE
DBCC FREESYSTEMCACHE('All')
DBCC DROPCLEANBUFFERS

USE master

-- 打开高级设置配置 
EXEC sp_configure 'show advanced options', 1
RECONFIGURE WITH OVERRIDE

-- 先设置物理内存上限到1G 
EXEC sp_configure 'max server memory (MB)', 1024
RECONFIGURE WITH OVERRIDE
"

sleep 10s

osql -E -S . -Q "
-- 还原原先的上限 
EXEC sp_configure 'max server memory (MB)', 25600
RECONFIGURE WITH OVERRIDE
"
```
