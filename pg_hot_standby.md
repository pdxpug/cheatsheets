Streaming rep + hot standby on Pg 9.3
=====================================

This is a checklist created and used by several members of [PDXPUG](http://pdxpug.wordpress.com) during a [lab day](http://pdxpug.wordpress.com/2014/01/30/pdxpug-streaming-rep-saturday-recap/).  We used Pg9.3 on a variety of linux systems.

Overview of steps
----------------
1. Set up your configs
2. Take your base backup of the master
3. Make sure you have the right configs on the standby
4. Start it up.

Relevant GUCs: master
-------------
* `max_wal_senders` >0, one for each standby, I think
* `wal_keep_segments`, dunno (we tried 16, our test VMs are low traffic)
* `wal_sender_timeout` just take the default
* `synchronous_standby_names` none, the default
* `vacuum_defer_cleanup_age` 0 (the default, leave it as-is)
* `archive_command`	- for the wal shipping part - point it somewhere *other than* pg_xlog, right? :) Something like `'cp %p /var/lib/postgresql/archive/%f'`
* `archive_mode` on - for the wal shipping part
* `wal_level` hot_standby - required if you have "hot standby" set to "on" on the standby

Relevant GUCs: standby
--------------
* `hot_standby` on
* `max_standby_archive_delay` leave as default
* `max_standby_streaming_delay` leave as default
* `wal_receiver_status_interval` leave as default
* `hot_standby_feedback` leave as default
* `wal_receiver_timeout` leave as default

Checklist
---------
1. Set up your configs.   Alter or create the following:
	```
	pg_hba.conf
	postgres.conf
	recovery.conf (only on the standby)
	```

	On the master:

	* change postgres.conf as noted above
	```
	wal_level = hot_standby
	archive_mode = on
	archive_command = 'cp %p /var/lib/postgresql/archive/%f'
	max_wal_senders = 3 #so we overdid it in our tests
	wal_keep_segments = 16
	```
	* restart
	* verify your changes (check both dirs)
	- `SELECT pg_switch_xlog();`
	* create your replication user:

	   `CREATE ROLE rbatty WITH LOGIN REPLICATION PASSWORD 'deckard';`
	   or
	   `CREATE USER rbatty WITH REPLICATION PASSWORD 'deckdard';`
	* change pg_hba.conf as required for access

	   `host   replication     rbatty          192.168.247.1/24        md5`
	- database name has to be "replication"!
	- reload

	On the standby:

	* change `postgres.conf` as noted above
	`hot_standby = on`
	* create `recovery.conf`
	```
	standby_mode = on
	primary_conninfo = 'host=192.168.247.181 port=5444 user=rbatty'
	restore_command = 'cp /var/lib/postgresql/archive/%f %p'
	```
	* create a `.pgpass` file for access from the master (an exercise left to the reader)

1.  Take your base backup of the master...let's try rsync  

	* copy the standby's .conf files somewhere they won't be overwritten
	* stop the master
	* copy master $PDATA over to standby, e.g.

	`rsync -av /path/to/pgdata/dir/* ip.of.standby.server:/path/to/pgdata/dir/`

	-a means archive, which preserves symbolic links, among other things.  
	-v is verbose.

	You can also use -e ssh option to rsync.

1.  You've now overwritten the configs on your standby;  copy them back from the safe place you stored them in the previous step.

1.  and start them up!  
(Does it matter which order?  Some say yes, some say no.)

Check if SR is working:  
----------------
1.  Check the process table:
 `ps -ef | grep wal`  
 On the master, there should be one wal sender for each standby:
 `postgres  4500  3792  2 13:45 ?        00:00:00 postgres: wal sender process rbatty 192.168.247.182[45067] streaming 2/98C836C8`

 On the standby, there should be one wal receiver:
 `postgres  3828  2524  3 13:45 ?        00:00:01 postgres: wal receiver process   streaming 2/98C836C8`

 Note that it's highly unlikely the wal segment name will match on a production instance.

1.  Check the status of the standby:  
 `SELECT pg_is_in_recovery();`  
 Should be 't'

1.  Check the status on the master:  
 `SELECT * FROM pg_stat_replication;`  
 This, combined with \watch, can be very educational.
