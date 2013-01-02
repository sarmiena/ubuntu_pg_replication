# PostgreSQL 9.1 Streaming Replication on Ubuntu Server 12.04 LTS 
This repo provides scripts and instructions to quickly setup PG replication on Ubuntu Server 12.04 LTS. It dives a 
little further than most how-tos by including cliffnotes on Christoffe Pettus' talks on PG replication 
(https://www.youtube.com/watch?v=k4f24zn5D4s). It also incorporates some default cluster and system tuning options to
avoid common problems.

# The quick and dirty
This setup was configured by creating 2 Ubuntu VM's on my laptop using VirtualBox. Assume the following:

* Master: 192.168.1.100 (hostname is 'pg')
* Slave: 192.168.1.101 (hostname is 'pgslave')

1. On both machines:

  ```
  administrator@pg:~$ sudo -s
  root@pg:~# apt-get install postgresql-9.1
  root@pg:~# su - postgres
  ```
2. Master:

  ```
  postgres@pg:~$ vi /etc/postgres/9.1/main/pg_hba.conf

    # Add line:
    host    replication     rep_user        192.168.1.101/32         md5

  postgres@pg:~$ psql
  postgres=# create user rep_user WITH REPLICATION PASSWORD 'seekrit';
  postgres=# \password
  postgres=# \q
  ```
  * Run pg_replication_config script (included in this repo)
  
  ```
  postgres@pg:~$ ./pg_replication_config --memory 2048 --file /etc/postgres/9.1/main/postgres.conf
  ```
3. Slave:

  ```
  postgres@pgslave:~$ pg_ctlcluster 9.1 main stop
  postgres@pgslave:~$ vi /etc/postgres/9.1/main/postgres.conf

    listen_addresses = '*'
    hot_standby = on

  postgres@pgslave:~$ vi /var/lib/postgres/9.1/main/recovery.conf

    standby_mode = on
    primary_conninfo = "host=192.168.1.100 port=5433 user=rep_user password=seekrit"
  ```
  * Run master_basebackup script
  ```
  postgres@pgslave:~$ master_basebackup 192.168.1.100
  ```
4. Test it out by creating a table on Master (via psql) & it should be streamed to Slave

# Important Notes (Highly recommended) 
* Based on Christophe Pettus' talk in Argentina: https://www.youtube.com/watch?v=k4f24zn5D4s
* (term) cluster: This term is somewhat confusing this day in age, but in the PostgreSQL world, 'cluster' refers to a single PostgreSQL instance running on a single box.
* initdb: creates a cluster (sadly initdb is named in direct conflict with the term above to produce more confusion)
  * creates files that will hold the cluster
  * does not start the server automatically
  * some package managers start after installation
* createdb: creates a single database
* pg_ctl: built in command to start and stop postgresql
  * init.d often uses pg_ctl
  * Ubuntu 12.04 LTS requires you to use pg_ctlcluster
* psql: command line interface for running postgresql querys, examine schema, etc.
* Postgresql directories
  * All data lives under a top-level directory
  * Let's call it $PGDATA
  * data lives in directory "base"
  * transaction logs live in pg_xlog (might be symbolic in some distros)
* Configuration files
  * Most installations, the config files live in $PGBASE
  * Debian-derived systems, they live in /etc/postgresql/9.2/main/...
  * postgresql.conf: most server settings
  * pg_hba.conf: who gets to log into the DB
* Users and Roles
  * A role is a database object that can own other objects (tables, etc), and that has privileges (able to write to a table). This is applied cluster-wide.
  * A user is just a role that can log into the system; otherwise, they're synonyms
  * Postgresql's security system is based around users
* Important parameters
  * Logging
  * Memory
  * Checkpoints
  * Planner
* Logging
  * Be generous with logging; it's very low impact on system.
  * It's your best source of information for finding performance problems.
  * Where to log?
    * syslog - if you have a syslog infrastructure you like already... then use it. Takes care of log rotation & stuffs.
      * LOCAL0.* ~/var/log/postgresql - prevents syslog from flushing after every single write
    * Otherwise, CSV format to files 
    * Do not use standard format; it's obsolete
  * What to log?
    log_destination             = 'csvlog'
    log_directory               = 'pg_log' 
    logging_collector           = on
    log_filename                = 'postgres-%Y-%m-%d_%H%M%S'
    log_rotation_age            = 1d
    log_rotation_size           = 10MB
    log_min_duration_statement  = 250ms # will log all queries that take longer that 250ms   
    log_checkpoints             = on
    log_connections             = on
    log_disconnections          = on
    log_lock_waits              = on
    log_temp_files              = 0 # logs any temp file
* Memory configuration
  * Shared memory follies
    * Postgresql allocates all shared memory at startup
    * Most Linux kernels don't allow much shared memory allocation
      * relevant Linux parameters:
        * SHMMAX: amount of shared shared memory you are allowed to allocate at once. 
        * SHMALL: amount of overall shared memory you are allowed to allocate across the whole system
      * To adjust SHMMAX and SHMALL:
        * value = shared_memory in bytes + 20%
        * sysctl -w kernel.shmmax = (value)
        * sysctl -w kernel.shmall = (value)/4096 #shmmll is in pages
  * shared_buffers
    * Below 2GB? - set to 20% of total system memory
    * Below 32GB - set to 25% of total system memory
    * Above 32GB - set to 8GB
  * OOM Killer Considered Harmful
    * The Linux OOM killer is a bug, not a feature, on Postresql servers
      * vm.overcommit_ration = 100
      * vm.overcommit_memory = 2
      * Swap = RAM
  * work_mem: 
    * how much memory postgres is allowed to use for a working operation (e.g. join table, sort table, etc).
    * if it cannot perform the operation with this amount of memory, then it performs the operation on disk (ouch).
    * does this for every operation in the entire system. e.g. if you have 23 joins going at the same time, then 23*work_mem is used.
    * Start low: 32-64MB
    * Tuning: 
      *Look for 'temporary file' lines in logs
      * Set to 2-3x the largest temp file you see. (extra padding because jobs need more memory than the size of the temp file to process work) -- this can lead to HUGE performance increases
      * But be careful: It can use that amount of memory per planner node.
  * maintenance_work_mem (amount of memory allowable for VACUUM)
    * 10% of system memory. up to 1 GB
    * Maybe even higher if you are having VACUUM problems.
  * effective_cache_size
    * this is a hint, not an allocation
    * set to the amount of file system cache available
    * if you don't know, set it to 50% of total system memory
* Linux
  * Turn off OOM killer
  * Use ext4 or XFS (whichever you prefer or whichever your distro prefers)
  * ext3 is not good for postgres servers
* WAL archiving 
  * Maintains a set of base backups and WAL segments on a (remote) server.
  * Can be used for a point-in-time recovery in case of an application (or DBA) failure.
  * Can be used along side streaming replication.
  * Debugging (using psql)
    * To find lock related issues: 
      * pg_stat_activity - all the connections that are open to the db & what they're doing. Flag that shows who is waiting for a lock.
      * pg_locks - who is waiting for the lock
* Checkpoints:
  1. Takes all the dirty buffers and writes them to disk
  2. Truncates the write ahead logs (WAL)

  * Potentially a lot of I/O
  * Done when the first two thresholds are hit:
    * A particular number of WAL segments have been written.
    * A timeout occurs
* Checkpoint settings:
  * WAL segments are roughly 16MB of changes (each in a file).
    * wal_buffers   = 16MB
    * checkpoint_completion_target = 0.9 # Spreads I/O over time (checkpoint_timeout*checkpoint_completion_target)
    * checkpoint_timeout = 10min  # Depends on restart time in case of crash only 
                                  # Bigger number results in less I/O, but longer *crash* restart times.
                                  # Rule of thumb for restart time is 20% of timeout (10m*0.2 == 2 minutes)
    * checkpoint_segments = 32 # To start.
  * Tuning: Look for checkpoint entries in the logs.. if they are happening more than checkpoint_timeout, then adjust checkpoint_segments so that checkpoints happen due to timeouts rather than filling segments.
  * Additional notes: 
    * The WAL can take up to 3 x 16MB x checkpoint_segments on disk
    * Restarting PG can take up to checkpoint_timeout (but usually less)
* Planner settings:
  * effective_io_concurrency: set to the number of I/O channels; otherwise, ignore it.
  * random_page_cost: planner has to make decisions about whether or not to scan index or scan entire table. This ratio represents how much faster is sequential access than random access. Default is 4.0 (too high).
    * 3.0 for typical RAID 10 array, 2.0 for SAN, 1.1 for Amazon EBS
  * Do not touch
    * fsync = on # never change this
    * synchronous_commit = on # Change this, but only if you understand that your WAL files might be missing transactions
* Changing settings
  * Most settings just require a server reload to take effect
  * Some require full server restart (such as shared_buffers).
  * Many can be set on a per-session basis!
* WAL (Write-Ahead Log), what is it good for?
  * Used to restore the database on an abnormal termination.
  * Absolutely essential to avoid data corruption.
  * The replay has to happen from the last consistent state.
    * = the last time a checkpoint finished
* Important WAL Facts.
  * It is the-ordered, so you can replay it to a particular point in time.
  * It is append-only, so it pays to put it on its own file system.
    * Not good to put on an SSD because constantly purged & SSD needs to be deleted before written.
  * It is the basis for both warm standby and streaming replication
* Indexing:
  * A good index:
    1. High selectivity (returns relatively low number of records compared to overall rows of table) AND is run frequently
    2. Is required to enforce a constraint (pks or uniqueness)
  * Bad index:
    * Anything not stated in 'Good index above'
    * If an index is rarely used, then it's not worth it because indexes are really expensive to maintain (on every insert).
    * Only the first column of a multi-column index can be used separately. If you index on A & B, and you query on B, that index won't help you.
  * Profiling:
    * pg_stat_user_tables - Shows sequential scans
    * pg_stat_user indexes - shows index usage
* Monitoring - Always monitor!
  * At least disk space
  * Memory and I/O utilization is very handy.
  * 1 minute bins
  * check_postgres.pl at bucardo.org (for monitoring).
  * also see OS utilities like iostat & df (also nagios for notifications)
* pg_dump
  * Everything works just fine... but can't do things like change schema.
  * Obv heavy on I/O
  * Creates a logical copy, not a physical disk copy
  * The copy is a snapshot of db the moment pg_dump was started
  * Gets impractical as the db gets larger (tens of GB)
* Streaming replication:
  * All or nothing - entire PG cluster needs to be replicated
  * You can use replicas for read-only queries
  * If you are getting query cancellations...
    * increase max_standby_streaming_delay to 200% of the longest query execution
  * You can pg_dump a streaming replica.
* MVCC: Multi Version Concurrency Control
  * Introduced by PG, now use by almost everyone.
  * Alternative to "pessemistic" locking strats.
  * Allows for much higher performance.
* MVCC rules.
  * Readers (to the same row) do not block readers.
  * Writers do not block readers - readers get old version of the row.
  * Readers do not block writes.
  * Writers *do* block writers to the same row.
* Versioning
  * Multiple versions of the same row can exist
  * Deleted and updated rows are not immediately removed from the database.
    * Some other transactions might be still be able to see them.
  * Solution: VACUUM?
* VACUUM
  * Scans each table for "dead" versions of tuples, and marks them as free.
  * Since 8.0, handled for you by the autovacuum daemon.
  * Good to manually vacuum after major update/delete operations.
* ANALYZE
  * The planner requires statistics on each table to make good guesses for how to execute queries
  * ANALYZE collections of these statistics
  * Done as part of VACUUM.
  * Always do it after a major database changes - especially a respore from a backup.
* Locking
  * Postgresql takes implicit locks on objects to maintain concurrency control.
  * Tuple locks on are on individual db rows.
  * Table, schema and db locks are on higher-level objects.
* Tuple locks
  * Share lock - prevents the row from being modified, but it can be read; any number of sessions can hold a shared lock on the same row.
  * Exclusive lock - prevents the row from being modified by anyone else; only one session can hold an exclusive lock.
* Surprising locks
  * Writing a dependent row can cause a share lock on the parent in a foreign key relationship
  * Updates on that parent row can then block
  * This is the reason for the fast/slow data rule.
* Table-level locks
  * Taken during schema modifications
  * Held only for as long as the schema modification goes on.
  * But this can be a very long time if you are adding a non-NULL column.
* Explicit locking
  * Taking an explicit lock on a table is a sign of an application problem.
  * If you think you can solve your problem with an explicit lock, no..
* A reminder about MVCC
  * All transactions see a snapshot of the database at the start of a transaction
  * Only writes to the same tuple (row) block
  * The transaction isolation levels control how "perfect" this snapshot model is.
* Replication
  * pg_basebackup - built in tool for doing a base backup to prime a secondary
  * repmgr.org

== Start ==

