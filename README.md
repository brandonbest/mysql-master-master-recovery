# mysql-master-master-recovery

Thank you to https://www.barryodonovan.com/2013/03/23/recovering-mysql-master-master-replication. I want to make sure I don't lose your notes.

MySQL Master-Master replication is a common practice and is implemented by having the auto-increment on primary keys increase by n where n is the number of master servers. For example (in my.conf):

This article is not about implementing this but rather about recovering from it when it fails. A work of caution – this former of master-master replication is little more than a useful hack that tends to work. It is typically used to implement hot stand-by master servers along with a VRRP-like protocol on the database IP. If you implement this with a high volume of writes; or with the expectation to write to both without application knowledge of this you can expect a world of pain!

It’s also essential that you use Nagios (or another tool) to monitor the slave replication on all masters so you know when an issue crops up.

So, let’s assume we have two master servers and one has failed. We’ll call these the Good Server (GS) and the Bad Server (BS). It may be the case that replication has failed on both and then you’ll have the nightmare of deciding which to choose as the GS!

You will need the BS to not process any queries from here on in. This may already be the case in a FHRP (e.g. VRRP) environment; but if not, use combinations of stopping services, firewalls, etc to stop / block access to the BS. It is essential that the BS does not process any queries besides our own during this process.
On the BS, execute STOP SLAVE to prevent it replicating from the GS during the process.
On the GS, execute:
STOP SLAVE; (to stop it taking replication information from the bad server);
FLUSH TABLES WITH READ LOCK; (to stop it updating for a moment);
SHOW MASTER STATUS; (and record the output of this);
Switch to the BS and import all the data from the GS via something like: mysqldump -h GS -u root -psoopersecret --all-databases  --quick  --lock-all-tables | mysql -h BS -u root -psoopersecret; Note that I am assuming that you are replicating all databases here. Change as appropriate if not.
You can now switch back to the GS and execute UNLOCK TABLES to allow it to process queries again.
On the BS, set the master status with the information your recorded from the GS via: CHANGE MASTER TO master_log_file='mysql-bin.xxxxxx', master_log_pos=yy;
Then, again on the BS, execute START SLAVE. The BS should now be replication from the GS again and you can verify this via SHOW SLAVE STATUS.
We now need to have the GS replicate from the BS again. On the BS, execute SHOW MASTER STATUS and record the information. Remember that we have stopped the execution of queries on the BS in step 1 above. This is essential.
On the GS, using the information just gathered from the BS, execute: CHANGE MASTER TO master_log_file='mysql-bin.xxxxxx', master_log_pos=yy;
Then, on the GS, execute START SLAVE. You should now have two way replication again and you can verify this via SHOW SLAVE STATUS on the GS.
If necessary, undo anything from step 1 above to put the BS back into production.
There is a --master-data switch for mysqldump which would remove the requirement to lock the GS server above but in our practical experience, there are various failure modes for the BS and the --master-data method does not work for them all.
