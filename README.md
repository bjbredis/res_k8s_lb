# res_k8s_lb


1. Set the proxy threads higher
   ```

   ```
   *note*: ensure all master shards proxy type
   ```
   rladmin bind db <db_name> endpoint <endpoint id> policy <all-master-shards|all-nodes>
   ```
2. Enabled distributed syncer on target DBs:


3. *ALREADY DONE* Set the replication buffer
	```
	ccs-cli hset bdb:6 internal_max_bulk_len 10737418240; rlutil dmc_reconf bdb=6
	rlutil config_set bdb=1 conf_keyword=client-query-buffer-limit conf_value=10G
	rlutil config_set bdb=1 conf_keyword=proto-max-bulk-len conf_value=10G

	rladmin tune db ecom-redb-m slave_buffer 2048
	```
4. Apply the patch and restart the syncer:
https://files.cs.redislabs.com/?/files/RLEC_Customers/81573/RLEC_Customers/81573/rhel7.zip

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
