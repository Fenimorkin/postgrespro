# postgrespro

#1 This SQL show you the relations buffered in database share buffer

The output ordered by relation percentage taken in shared buffer. It also shows that how much of the whole relation is buffered.

#select c.relname,pg_size_pretty(count(*) * 8192) as buffered, 
        round(100.0 * count(*) / ( 
           select setting from pg_settings 
           where name='shared_buffers')::integer,1)
        as buffer_percent, 
        round(100.0*count(*)*8192 / pg_table_size(c.oid),1) as percent_of_relation
from pg_class c inner join pg_buffercache b on b.relfilenode = c.relfilenode inner 
join pg_database d on ( b.reldatabase =d.oid and d.datname =current_database()) 
group by c.oid,c.relname order by 3 desc limit 10;

        relname         | buffered | buffer_percent | percent_of_relation
------------------------+----------+----------------+---------------------
 t_acl                  | 1869 MB  |           60.8 |               100.0
 t_inodes_pkey          | 128 MB   |            4.2 |                 9.8
 t_dirs                 | 129 MB   |            4.2 |                 3.3
 t_locationinfo_pkey    | 95 MB    |            3.1 |                 3.6
 i_t_acl_rs_id          | 96 MB    |            3.1 |                 8.3
 i_dirs_ipnfsid         | 86 MB    |            2.8 |                 5.1
 t_dirs_pkey            | 85 MB    |            2.8 |                 3.2
 t_inodes_checksum_pkey | 74 MB    |            2.4 |                 6.5
 t_inodes               | 64 MB    |            2.1 |                 2.1
 t_tags_pkey            | 63 MB    |            2.1 |                 6.1



#2 relation usage count in PostgreSQL database shared buffer

#select c.relname,count(*) as buffers,usagecount
 from pg_class c
      inner join pg_buffercache b on b.relfilenode = c.relfilenode 
      inner join pg_database d on (b.reldatabase = d.oid and d.datname =current_database()) 
  group by c.relname,usagecount 
  order by c.relname,usagecount;

relname        | buffers | usagecount

---------------+---------+------------
i_dirs_iparent | 1047    | 0
i_dirs_iparent | 1620    | 1
i_dirs_iparent | 471     | 2
i_dirs_iparent | 389     | 3
i_dirs_iparent | 194     | 4
i_dirs_iparent | 110     | 5



#3 disk usage

select nspname,relname,pg_size_pretty(pg_relation_size(c.oid)) as "size" 
from pg_class c left join pg_namespace n on ( n.oid=c.relnamespace) 
where nspname not in ('pg_catalog','information_schema') 
order by pg_relation_size(c.oid) desc limit 30;

 nspname |          relname          |  size
---------+---------------------------+---------
 public  | t_access_latency          | 5263 MB
 public  | t_locationinfo            | 3919 MB
 public  | t_dirs                    | 3880 MB
 public  | t_inodes                  | 3087 MB
 public  | t_level_2                 | 2906 MB
 public  | t_dirs_pkey               | 2683 MB
 public  | t_locationinfo_pkey       | 2678 MB



#4 top relation in cache

select c.relname,count(*) as buffers 
from pg_class c 
     inner join pg_buffercache b on b.relfilenode=c.relfilenode 
     inner join pg_database d on (b.reldatabase=d.oid and d.datname=current_database()) 
group by c.relname order by 2 desc limit 20;
         relname         | buffers
-------------------------+---------
 t_acl                   |  239219
 t_inodes_pkey           |   16417
 t_dirs                  |   16391
 i_t_acl_rs_id           |   12160
 t_locationinfo_pkey     |   11666
 t_dirs_pkey             |   11223
 i_dirs_ipnfsid          |   10736
 t_inodes_checksum_pkey  |    9231



#5 summary of buffer usage count

# select usagecount,count(*),isdirty 
  from pg_buffercache 
  group by isdirty,usagecount 
  order by isdirty,usagecount;
 usagecount | count  | isdirty
------------+--------+---------
          0 |  24075 | f
          1 | 262086 | f
          2 |  23744 | f
          3 |  23788 | f
          4 |  24135 | f
          5 |  19386 | f
          1 |   1971 | t
          2 |   1206 | t
          3 |   1128 | t
          4 |   4209 | t
          5 |   7488 | t




#6 lock information in postgreSQL(pg9.1)

#select locktype,virtualtransaction,transactionid,nspname,relname,mode,granted,
        cast(date_trunc('second',query_start) as timestamp) as query_start,
        substr(current_query,1,60) as query 
from pg_locks 
     left outer join pg_class on (pg_locks.relation = pg_class.oid) 
     left outer join pg_namespace on (pg_namespace.oid = pg_class.relnamespace), pg_stat_activity
where not pg_locks.pid=pg_backend_pid() and pg_locks.pid = pg_stat_activity.procpid 
order by virtualtransaction;





#7 lock information in postgreSQL(pg9.2,pg9.3)

# select locktype,virtualtransaction,transactionid,nspname,relname,mode,granted,
        cast(date_trunc('second',query_start) as timestamp) as query_start,
        substr(query,1,60) as query 
from pg_locks 
     left outer join pg_class on (pg_locks.relation = pg_class.oid) 
     left outer join pg_namespace on (pg_namespace.oid = pg_class.relnamespace), pg_stat_activity
where not pg_locks.pid=pg_backend_pid() and pg_locks.pid = pg_stat_activity.pid 
order by virtualtransaction;
   locktype    | virtualtransaction | transactionid | nspname |      relname       |       mode       | granted |     query_start     | query  
---------------+--------------------+---------------+---------+--------------------+------------------+---------+---------------------+--------
 relation      | 37/63787           |               | public  | t_retention_policy | RowExclusiveLock | t       | 2014-07-22 22:06:17 | COMMIT
 virtualxid    | 37/63787           |               |         |                    | ExclusiveLock    | t       | 2014-07-22 22:06:17 | COMMIT
 transactionid | 37/63787           |      69587764 |         |                    | ExclusiveLock    | t       | 2014-07-22 22:06:17 | COMMIT
