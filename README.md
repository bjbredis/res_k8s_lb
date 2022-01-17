# res_k8s_lb


1. Set the proxy threads higher
   ```
	tune proxy all threads 8
	tune proxy all max_threads 16
   ```
   *note*: ensure all master shards proxy type for the primary DB
   ```
   rladmin bind db <db_name> endpoint <endpoint id> policy <all-master-shards|all-nodes>
   ```
2. Enabled distributed syncer on target DBs:

	```
	tune db <db_name> syncer_mode distributed
	```

3. Set the replication buffer
	```
	ccs-cli hset bdb:6 internal_max_bulk_len 10737418240; rlutil dmc_reconf bdb=6

	rlutil config_set bdb=1 conf_keyword=client-query-buffer-limit conf_value=10G
	rlutil config_set bdb=1 conf_keyword=proto-max-bulk-len conf_value=10G

	rladmin tune db ecom-redb-m slave_buffer 2048
	rladmin tune db ecom-redb-s1 slave_buffer 10240
	rladmin tune db ecom-redb-s2 slave_buffer 10240
	```
4. Apply the patch and restart the syncer:
https://files.cs.redislabs.com?/share/view/2tbqp-4xnrcdvb

But for this patch, you only need the syncer binary which is the file `rhel7\redislabs-100.0.5-5029.rhel7.x86_64.rpm\redislabs-100.0.5-5029.rhel7.x86_64.cpio\.\opt\redislabs\bin\rl_syncer`
The old \opt\redislabs\bin\rl_syncer in the cluster nodes needs to be replaced with this rl_syncer file.  Once you've replaced this file on all nodes, you must also restart the syncers.
```
curl -v -k -u <USERNAME>:<PASSWORD> -X PUT -H "Content-Type: application/json" -d '{"replica_sync":"disabled"}' https://localhost:9443/v1/bdbs/7
curl -v -k -u <USERNAME>:<PASSWORD> -X PUT -H "Content-Type: application/json" -d '{"replica_sync":"disabled"}' https://localhost:9443/v1/bdbs/8
```

```
curl -v -k -u <USERNAME>:<PASSWORD> -X PUT -H "Content-Type: application/json" -d '{"replica_sync":"enabled"}' https://localhost:9443/v1/bdbs/7
curl -v -k -u <USERNAME>:<PASSWORD> -X PUT -H "Content-Type: application/json" -d '{"replica_sync":"enabled"}' https://localhost:9443/v1/bdbs/8
```
Normally we would backup the original file in case the patch does not work.  Since this is in k8s environment, we can always restart the nodes (one by one) and the original file will be used again so no need to backup before replace.
As for validating step, I think you can use the same setup you did to reproduce the syncer restart with multiple WAIT commands.

```
oc cp rl_syncer rec-replica-1:/opt/redislabs/bin/rl_syncer; 
oc rsh rec-replica-1 md5sum /opt/redislabs/bin/rl_syncer; 
oc rsh rec-replica-1 supervisorctl restart dmcproxy;
```