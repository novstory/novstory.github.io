---
layout: post
title: "[MySQL] Crashing with add/drop column with ALGORITHM=INSTANCE"
date: 2023-03-22
tags: [MySQL]
---

MySQL Server 8.0.29 버전 이후부터 Column Add/Drop 처리 시 기본적으로 ALGORITHM=INSTANT 처리 되도록 변경되었습니다. 이 기능이 적용 됨에 따라 몇몇 Side Effect들이 심심치 않게 확인되고 있는데요. Column Add/Drop 처리 실패 및 Instance Crash, Percona XtraBackup 실패 등이 그 버그 사례들입니다. 이번 장에서는 해당 버그들의 사례들을 일부 재현해보고 개선된 버전이 있는지 같이 살펴보도록 하겠습니다.

### Column Add/Drop 처리 실패

```sql
# MySQL 8.0.30 버전에서 테스트
[root]# mysql -V
mysql  Ver 8.0.30 for Linux on x86_64 (MySQL Community Server – GPL)

mysql> create database test;

mysql> use test;

mysql>
create table `test_table_1`
(
  `id`       bigint         unsigned not null auto_increment primary key,
  `seq`      int(11)        not null,
  `url1`     varchar(1000)  default null,
  `url2`     varchar(1000)  default null,
  `url3`     varchar(1000)  default null,
  `reg_id`   varchar(20)    default null,
  `reg_dttm` timestamp null default null,
  key `seq` (`seq`)
) engine=innodb default charset=utf8mb4 collate=utf8mb4_general_ci;

mysql> show create table test_table_1\G

# 데이터 10만건 적재
mysql>
delimiter $$
drop procedure if exists datainsert100000$$
create procedure datainsert100000()
begin
declare i int default 1;
while(i < 100001) do
     insert into test_table_1(seq, url1, url2, url3, reg_id, reg_dttm) values (1234, 'http://m.test.com/index.jsp?infoMobileAccessYn=G1&Info=1234_5678&url=ABCD.web', NULL, NULL, NULL, now());
     set i = i + 1;
end while;
end$$
delimiter $$

mysql> start transaction;
mysql> call datainsert100000();
mysql> commit;

mysql> select name, total_row_versions from information_schema.innodb_tables where total_row_versions > 0 order by total_row_versions desc;
Empty set (0.00 sec)

# MySQL 8.0.29 버전부터 algorithm=instant 값이 기본 적용됨
mysql> alter table test_table_1 add column instance varchar(100) default 'abcd';
mysql> alter table test_table_1 drop column instance;

mysql> select name, total_row_versions from information_schema.innodb_tables where total_row_versions > 0 order by total_row_versions desc;
+-------------------+--------------------+
| name              | total_row_versions |
+-------------------+--------------------+
| test/test_table_1 |                  2 |
+-------------------+--------------------+

# 62번 더 수행
mysql> alter table test_table_1 add column instance varchar(100) default 'abcd';
mysql> alter table test_table_1 drop column instance;
...

# Maximum row versions permitted is 64, DDL은 64개까지 허용됨
mysql> select name, total_row_versions from information_schema.innodb_tables where total_row_versions > 0 order by total_row_versions desc;
+-------------------+--------------------+
| name              | total_row_versions |
+-------------------+--------------------+
| test/test_table_1 |                 64 |
+-------------------+--------------------+

mysql> alter table test_table_1 add column instance varchar(100) default 'abcd';
ERROR 4092 (HY000): Maximum row versions reached for table test/test_table_1. No more columns can be added or dropped instantly. Please use COPY/INPLACE.

```

total_row_versions 값이 허용 최대치인 64에 도달했을 때 더이상 Instant DDL 구문 처리가 되지 않는 것을 확인 할 수 있습니다. 추후 버전에서는 64에 도달했을 경우 자동으로 ALGORITHM=COPY/INPLACE 처리 되도록 개선은 되었습니다.



그럼 Percona XtraBackup 실패 사례도 한번 살펴볼까요? 어느 순간 백업이 실패하고 있다면 이 버그를 의심해 볼 만한 상황일 겁니다.

### Percona XtraBackup 실패

```shell
# MySQL 8.0.30 버전에서 테스트
[root]# mysql -V
mysql  Ver 8.0.30 for Linux on x86_64 (MySQL Community Server – GPL)

# Percona XtraBackup 8.0.30 버전에서 테스트
[root]# xtrabackup --version
xtrabackup version 8.0.30-23 based on MySQL server 8.0.30 Linux (x86_64) (revision id: 873b467185c)

[root]# xtrabackup --defaults-file=/etc/my.cnf --backup  --user=mysql_admin -p --target-dir=/MYSQL_BKUP/`date +%Y%m%d`
...
2023-03-13T13:38:53.160841+09:00 0 [Note] [MY-011825] [Xtrabackup] Connecting to MySQL server host: localhost, user: mysql_admin, password: set, port: 3306, socket: /tmp/mysql.sock
2023-03-13T13:38:53.170626+09:00 0 [Note] [MY-011825] [Xtrabackup] Using server version 8.0.30
2023-03-13T13:38:53.175107+09:00 0 [Note] [MY-011825] [Xtrabackup] Executing LOCK INSTANCE FOR BACKUP ...
2023-03-13T13:38:53.177862+09:00 0 [ERROR] [MY-011825] [Xtrabackup] Found tables with row versions due to INSTANT ADD/DROP columns
2023-03-13T13:38:53.177907+09:00 0 [ERROR] [MY-011825] [Xtrabackup] This feature is not stable and will cause backup corruption.
2023-03-13T13:38:53.177920+09:00 0 [ERROR] [MY-011825] [Xtrabackup] Please check https://docs.percona.com/percona-xtrabackup/8.0/em/instant.html for more details.
2023-03-13T13:38:53.177932+09:00 0 [ERROR] [MY-011825] [Xtrabackup] Tables found:
2023-03-13T13:38:53.177940+09:00 0 [ERROR] [MY-011825] [Xtrabackup] test/test_table_1
2023-03-13T13:38:53.177951+09:00 0 [ERROR] [MY-011825] [Xtrabackup] Please run OPTIMIZE TABLE or ALTER TABLE ALGORITHM=COPY on all listed tables to fix this issue.
```

MySQL Server 8.0.29 버전부터 기본 적용된 Instant DDL 처리 방식으로 인한 버그 사례들은 이외에도 다양합니다.

### MySQL 8.0.29 버전 이후 Instant DDL 결함사항들

- Bug 34233264 - Assertion failure: rec.h:834:!rec_new_is_versioned(rec)
- Bug 34243694 - DROP COLUMN with ALGORITHM=INSTANT causing sporadic table corruption
- Bug 34181432 - Assert: rem0lrec.h:314:len == rec_get_nth_field_size_low(rec, n - n_drop)
- Bug 34302445 - Assertion failure: dict0dd.cc:1693:dd_column_is_dropped(old_col), Fixed as of the upcoming 8.0.30 release
- Bug 34488482 - Assertion failure: fil0fil.cc:7533 thread 140697559873280, Fixed as of the upcoming 8.0.31 release

최근 Bug 34488482의 경우 8.0.31 버전이 되서야 Fix된 것을 알 수 있습니다.

[Error Message: Found tables with row versions due to INSTANT ADD/DROP columns](https://docs.percona.com/percona-xtrabackup/8.0/em/instant.html)

<figure>
<img src="/assets/img/percona_xtrabackup.jpg" alt="XtraBackup">
<figcaption>Percona XtraBackup</figcaption>
</figure>



참고로 최근 Percona 측에서 Instant DDL 관련 개선 버전을 출시하였습니다. MySQL Server 8.0.32, Percona XtraBackup 8.0.32-25 버전 사용을 권장하고 있습니다.

> > [Percona XtraBackup 8.0.32-25 Download](https://www.percona.com/downloads)

> > [MySQL Community Server 8.0.32 Download](https://dev.mysql.com/downloads/file/?id=516816)

