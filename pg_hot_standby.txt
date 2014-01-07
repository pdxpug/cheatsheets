Streaming rep + hot standby on Pg 9.3

Gleaned from the latest docs:
On the master:
max_wal_senders >0, say 5
wal_keep_segments, dunno (how about 16, this is low traffic)
wal_sender_timeout just take the default
synchronous_standby_names (none, the default - this is disabled & I don't need it)
vacuum_defer_cleanup_age 0 (the default, leave it as-is)
archive_command	- for the wal shipping part - point it somewhere *other than* pg_xlog, right? :)
archive_mode on - for the wal shipping part
wal_level hot_standby (I think)

Note:  the comments in the config file have really improved since v9.

On the standby:
hot_standby on
max_standby_archive_delay leave as default
max_standby_streaming_delay leave as default
wal_receiver_status_interval leave as default
hot_standby_feedback I want this on to see what it does
wal_receiver_timeout leave as default

I think the steps are as follows (this process has changed, and the docs don't really have a step-by-step "do [x] if you want [y]" procedure. :P )

1. Set up your configs:
	- Set up continuous archiving on the master (archive must be accessible from the standby.)
2. Take your base backup of the master, *from the standby server* (see your recovery cheatsheet)
3. Make sure you have the right config on the standby
4. Start it up.

1. Files I have to tinker with:
pg_hba.conf
postgres.conf
recovery.conf (only on the standby)

master:
change postgres.conf as noted above
	- restart
	- SELECT pg_switch_xlog(); to verify (check both dirs)
	- CREATE ROLE rbatty WITH REPLICATION PASSWORD 'deckard';
change pg_hba.conf as required for access
	- database name has to be replication!
	- reload

standby:
change postgres.conf as noted above
check recovery.conf
create a .pgpass file for access from the master

2.  backup...let's try rsync
Copy the standby's .conf files somewhere they won't be overwritten

stop the master
then on the master server, use rsync:
rsync -av /path/to/pgdata/dir/* ip.of.standby.server:/path/to/pgdata/dir/

-a means archive, which preserves symbolic links, among other things.
-v is verbose.

You can also use -e ssh

3.  check your configs on your standby.

4.  and start them up!
standby first
then master

Check if SR is working:
ps -ef | grep wal
On the master, there should be one wal writer, then one wal sender for each standby
On the standby, there should be one wal receiver

On the standby:
SELECT pg_is_in_recovery();
Should be 't'
