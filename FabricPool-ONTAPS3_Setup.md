## Setting Up FabricPool Client To Tier to ONTAP S3

#### Notes
* S3 LIF on the S3 Object Store Server cannot have any other data services on the policy except for data-s3-server and data-core-services.
* FabricPool client connects to the S3 LIF on the destination S3 SVM using the intercluster LIFS on the Admin SVM.
   * Admin SVM needs to be able to resolve DNS and verify HTTPS Certificate of the S3 Server.
* Intercluster LIFs are only required on the FabricPool client.  If you want to do S3 mirroring, then intercluster LIFs required on S3 Object Store Server.
* When running the profiler, it automatically deletes objects after it’s done
* Cannot change aggr-multiplier after creating Bucket (default is 4)
* Max constituent volume size is 300TB, minimum constituent volume size is 100GB, in 9.16.1 it is now 50GB
* Maximum number of constituent members is 200
* Max bucket size = Max flexgroup constituents is 200.  Max constituent size is 300TB, so around 60PB total.
* Regarding constituents, you cannot get better performance than 16 per node/32 per HA pair. Everything after that is exclusively for capacity.
* To prevent performance bottlenecks with multiple buckets on single object-store-server, it is best to create a separate object-store-server for each bucket.  At a minimum, best is to have a separate bucket for each cluster.
* Beware of over provisioning, make sure to set up proper monitoring
* Need to obtain the following Certificates - Root CA, intermediary CA, S3 Object Store Server Public and Private Key
* The certificate name is used by the S3 object-store-server.  When issuing new certificates, need to install the new certificate on the S3 Cluster and then need to set the object-store-server to use the new certificate.
* On the Fabricpool client, only need to install the Root and intermediary CA certs.  Only need to update them when they expire.  The FabricPool client validates the certificates with the Intermediary and Root CA servers.
* When to tier data depends on both the aggregate tiering threshold and volume tiering settings.  Aggregate tiering threshold takes precedence and it will only tier data after the threshod has been exceeded and it will only tier until it gets below the tiering threshold.
* FlexGroup and FlexVol volumes residing on an aggregate must be configured for thin provisioning before a cloud tier can be attached to the aggregate.
* When the blocks comprising a file's contents get moved out to the cloud tier, the file's WAFL metadata remains in the local tier.
* Cloud retrieval, if set to auto, will retrieve random reads (not sequential), will not retrieve if aggregate is greater than 70% full.  It will mark the data as hot and tier it back during the next scheduled retrieval.
* If using Snapshot only tiering policy, the Snapshot will only get tiered to the capacity tier if the blocks in the Snapshot no longer exist in the active file system.
* Enable Inactive Data Reporting on the Aggregate to begin the cooling period countdown so that tiering can start once it is attached to bucket.

#### Create SVM for the S3 Object Store Server
```
vserver create -vserver s3svm -rootvolume s3svm_rootvol -aggregate nodeAaggr1 -data-services data-s3-server -rootvolume-security-style unix -language en_US -snapshot-policy default  
vserver services dns create -vserver s3svm -domains company.com -state enabled -name-servers x.x.x.x,x.x.x.x -skip-config-validation true  
unix-user modify -vserver s3svm -user root -primary-gid 0  
vol modify -volume  s3svm_rootvol -group root -unix-permissions 701  
vserver add-aggregates -vserver s3svm -aggregates nodeAaggr1,nodeAaggr2,nodeBaggr1,nodeBaggr2  
vserver remove-protocols -vserver s3svm -protocols nfs,cifs,ndmp,iscsi,fcp  
network interface service-policy create -vserver s3svm -policy data-s3-server -services data-core,data-s3-server  
route create -vserver s3svm -destination 0.0.0.0/0 -gateway x.x.x.x
route create -vserver adminsvm -destination 0.0.0.0/0 -gateway x.x.x.x 
net int create -vserver s3svm -lif nodeA_s3 -service-policy data-s3-server -home-node nodeA -home-port e0a -address nodeAs3_x.x.x.x -netmask x.x.x.x -auto-revert false  
net int create -vserver s3svm -lif nodeB_s3 -service-policy data-s3-server -home-node nodeB -home-port e0a -address nodeBs3_x.x.x.x -netmask x.x.x.x -auto-revert false  
net int create -vserver adminsvm -lif nodeA_icl -role intercluster -home-node nodeA -home-port e0a -address nodeAicl_x.x.x.x -netmask x.x.x.x  
net int create -vserver adminsvm -lif nodeB_icl -role intercluster -home-node nodeB -home-port e0a -address nodeBicl_x.x.x.x -netmask x.x.x.x
```


#### Configure the S3 Object Store Server
**Install the Certificates**  
```
security certificate install -type server -vserver s3svm  
- Copy/paste the public key, copy from BEGIN to END, hit enter twice
- Copy/paste the private key, copy from BEGIN to END, hit enter twice
- Do you want to continue entering root and/or intermediate certificates {y|n}: No  
```	
```
security certificate install -type server-ca -vserver s3svm 
- Copy/paste the certificate key for the Root CA; enter No when prompted to install another Certificate  
```
```
security certificate install -type server-ca -vserver s3svm  
- Copy/paste the certificate key for the Intermediary CA; enter No when prompted to install another Certificate
```
```
security certificate show-user-installed
```  

**Configure Object Store Server and Bucket**  
```
vserver object-store-server create -vserver s3svm -object-store-server s3vm-s3.xx.company.com -certificate-name s3vm-s3.xx.company.com -secure-listener-port 443 -is-http-enabled false -is-https-enabled true 
network connections listening show -vserver s3svm 
vserver object-store-server user create -vserver s3svm -user fabricpoolsvm-s3-user
object-store-server user show -user fabricpoolsvm-s3-user -> Save the access and secret keys in a secure place.  
vserver object-store-server bucket create -vserver s3svm -bucket bucket-s3-fabricpoolsvm  -aggr-list nodeAaggr1,nodeAaggr2,nodeBaggr1,nodeBaggr2 -aggr-list-multiplier 8 -size 2PB  
vserver object-store-server bucket show  
vserver object-store-server bucket policy add-statement -vserver s3svm -bucket bucket-s3-fabricpoolsvm -effect allow -action GetObject,PutObject,DeleteObject,ListBucket,GetBucketAcl,GetObjectAcl,ListBucketMultipartUploads,ListMultipartUploadParts -principal fabricpoolsvm-s3-user -resource bucket-s3-fabricpoolsvm,bucket-s3-fabricpoolsvm/*
- This allows full access to bucket and its objects for specific user.
volume modify -vserver s3svm -volume bucket-s3-fabricpoolsvm -snapshot-policy none -percent-snapshot-space 0
```

**Additional Commands For Object Store Server**
```
vserver object-store-server users regnerate-keys -user fabricpoolsvm-s3-user -vserver s3svm
- If lose keys, can regenerate but will need to update the FabricPool client too.  

vserver object-store-server bucket policy-statement-condition create -vserver s3svm -bucket bucket-s3-s3svm -index 1 -operator ip-address -source-ips <ic-ip1>, <ic-ip2>, etc. 
- If planning to attach other SVMs to the object store, then should set statement policy permissions to only allow data SVM access to that particular bucket.  For source-IPs, need to include the intercluster LIF IP of every node in the fabric pool client cluster and need to keep it up to date.
```

#### Configure the Fabric Pool Client
```
security certificate install -type server-ca -vserver clustersvm 
- Copy/paste the certificate key for the Root CA; enter No to install another Certificate  
security certificate install -type server-ca -vserver clustersvm
- Copy/paste the certificate key for the Intermediary CA; enter No to install another Certificate  

storage aggregate object-store config create -object-store-name s3svm-s3 -provider-type ONTAP_S3 -server s3vm-s3.xx.company.com -container-name bucket-s3-fabricpoolsvm -is-ssl-enabled true -is-certificate-validation-enabled true -access-key <key> -secret-password <password> 
storage aggregate object-store config show -instance  
- verify path-style, 443, truefor ssl/tls cert validation enabled, true for tiering allowed, none for encryption of data at rest by the object store server  

aggregate object-store attach -aggregate aggr_noflexgroup -object-store-name s3svm-s3 -check-only true - No FlexGroup  
aggregate object-store attach -aggregate aggr_flexgroup -object-store-name s3svm-s3 -allow-flexgroup true -check-only true - FlexGroup

*If you see this warning, then check the volumes efficient settings*
*To avoid warning, the volumes efficiency setting should be default.  If your volumes workload is performance intensive, then move the volume to a non-FabricPool aggregate.
Warning: Enabling FabricPool on an aggregate that contains auto adaptive compressed volumes is not recommended for performance-critical applications.
volume efficiency show -storage-efficiency-mode efficient

aggregate object-store attach -aggregate aggr_noflexgroup -object-store-name s3svm-s3 - No FlexGroup  
aggregate object-store attach -aggregate aggr_flexgroup -object-store-name s3svm-s3 -allow-flexgroup true - FlexGroup

aggregate object-store show -fields tiering-fullness-threshold (default is 50%) - it will only start tiering once aggr is above 50% and tier until it gets down to 50%
volume modify -volume volname -tiering-policy auto -tiering-minimum-cooling-days 31 (default are auto, 31)

storage aggregate object-store profiler start -object-store-name s3svm-s3 -node fabricpoolnode  
storage aggregate object-store profiler show  
storage aggregate object-store show-space  
```

**Additional Commands For FabricPool Client**
```
- To expand the bucket, expand the underlying flexgroup and then modify bucket size  
volume expand -vserver fabricpoolsvm -volume fg_name -aggr-list aggregate name,... [-aggr-list-multiplier constituents_per_aggr] 
vserver object-store-server bucket modify -vserver <svm name> -bucket <bucket name> -size <new size>
```

#### Configure EMS Alerts on both the FabricPool Client and S3 Object Store Server
```
- Volume Full Alerts 
event filter create -filter-name volumefull    
event filter rule add -filter-name volumefull -type include -message-name monitor.volume.nearlyFull    
event filter rule add -filter-name volumefull -type include -message-name monitor.volume.Full    
event filter rule add -filter-name volumefull -type include -message-name wafl.vol.runningOutOfInodes    
event filter rule add -filter-name volumefull -type include -message-name wafl.vol.full    
event config modify -mail-server smtp.mail.com -mail-from cluster@mail.com    
event notification destination create -name user1 -email user1@mail.com    
event notification destination create -name user2 -email user2@mail.com    
event notification destination show    
event notification create -filter-name volumefull -destinations user1,user2    
event filter show -filter-name volumefull    
event filter test -filter-name volumefull
```

```
- FlexGroup Full Alerts
event filter create -filter-name fgconstituentfull    
event filter rule add -filter-name fgconstituentfull -type include -message-name fg.inodes.member.full    
event filter rule add -filter-name fgconstituentfull -type include -message-name fg.inodes.member.nearlyFull   
event filter rule add -filter-name fgconstituentfull -type include -message-name fg.space.member.full  
event filter rule add -filter-name fgconstituentfull -type include -message-name fg.space.member.nearlyFull  
event config modify -mail-server smtp.mail.com -mail-from cluster@mail.com    
event notification destination create -name user1 -email user1@mail.com  
event notification destination create -name user2 -email user2@mail.com   
event notification destination show    
event notification create -filter-name fgconstituentfull -destinations user1,user2   
event filter show -filter-name fgconstituentfull
```

```  
- Aggr Full Alerts
event filter create -filter-name aggrfull   
event filter rule add -filter-name aggrfull -type include -message-name vol.phys.overalloc   
event config modify -mail-server smtp.mail.com -mail-from cluster@mail.com    
event notification destination create -name user1 -email user1@mail.com    
event notification destination create -name user2 -email user2@mail.com   
event notification destination show  
event notification create -filter-name aggrfull -destinations user1,user2    
event filter show -filter-name aggrfull
```

```
- Object Store and Certificate Alerts
event filter create -filter-name objcerts    
event filter rule add -filter-name objcerts-type include -message-name object.store.full   
event filter rule add -filter-name objcerts-type include -message-name ktls.failed    
event filter rule add -filter-name objcerts-type include -message-name ktls.cnxnHandshakeLimit    
event filter rule add -filter-name objcerts-type include -message-name mgmtgwd.certificate.expiring    
event filter rule add -filter-name objcerts-type include -message-name mgmtgwd.certificate.expired   
event filter rule add -filter-name objcerts-type include -message-name objstore.interclusterlifDown   
event filter rule add -filter-name objcerts-type include -message-name object.store.unavailable   
event filter rule add -filter-name objcerts-type include -message-name wafl.ca.latency.threshold     
event filter rule add -filter-name objcerts-type include -message-name s3.bucket.invalid.accessKey   
event filter rule add -filter-name objcerts-type include -message-name objstore.host.unresolvable  
event config modify -mail-server smtp.mail.com -mail-from cluster@mail.com      
event notification destination create -name user1 -email user1@mail.com     
event notification destination create -name user2 -email user2@mail.com      
event notification destination show    
event notification create -filter-name objcerts -destinations user1,user2    
event filter show -filter-name objcerts
```

#### Renewing Certificates
* Object Store Server Certificate only needs to be updated on the S3 server.  If Intermediary and/or Root Certificates are expiring, then need to update these Certificates on both FabricPool client and S3 server.  
```
- On S3 Object Store Server
security certificate rename -vserver s3svm -cert-name s3svm.domain.com -new-name s3svm.domain.com.old  
security certificate install -type server -vserver s3svm -name cert-name s3svm.domain.com  
vserver object-store-server modify -vserver s3svm -object-store-server s3svm.domain.com -certificate-name s3svm.domain.com  

- On FabricPool Client  
storage aggregate object-store profiler start -object-store-name <name> -node <nodename>  
storage aggregate object-store-profiler show  
event log show -message-name ktls.*	

- If issues with Certificates, run the following command to disable certificate verification:
storage aggregate object-store config modify -object-store-name <name> -is-certificate-validation-enable false
```
			 
#### Commands for Monitoring Performance
```
qos statistics volume latency show*** - check the Cloud column for latency
node run -node nodename -command sysstat 1
set diag; node run -node * "wafl composite stats show"
set diag; node run -node * "wafl composite stats counter show FPAGGR"
set diag; node run -node * "wafl cloudio_stats"
set diag; statistics show -object object_store_client_conn -instance *
set diag; statistics show -object object_store_client_op -instance * -raw
statistics show -object ktls_global -instance ktls_global -raw
```

#### Things That Could Create Performance Issues
* Security scanners like Windows AV, man-in-the-middle device such as WAF or DEP that are reading data stored in a FabricPool can pull large amounts of data from the capacity tier (cold storage) causing resource contention on the node.

#### Troubleshooting Tiering Issues
```
- Check the tiering policy.  It could be that there isn't any inactive data based on the time specified in the tiering policy.  Tiering policy needs to be set to none for FabricPool attached aggregates to be able to report inactive data.  Changing the tiering policy to none will report the amount of the entire volume that is inactive for at least 31 days.  For the other policies, inactive data will not be reported.
volume modify -vserver svm -volume volumename -tiering-policy none
volume show -vserver svm -volume volumename -fields  performance-tier-inactive-user-data,performance-tier-inactive-user-data-percent
	
- Manually trigger the scan.  This command triggers a tiering and retrieve scan. Tiering and retrieval behavior is driven by the tiering and cloud retrieve policy settings on the volume. The cloud retrieval policy must be set to promote to enable scanner based retrieval.
volume object-store tiering trigger
vserver object-store-server bucket show -fields object-count -> run this on the object store server
volume object-store tiering show
storage aggregate object-store show -fields object-store-availability,object-store-unavailable-reason

- Check EMS messages to see if there are any tiering error messages
event log show -messagename fp.est.scan.catalog.updated -nodename nodename
event log show -messagename wafl.scan.* -nodename nodename -event *volname*

- Can try changing the tiering policy to all and then trigger the tiering command.  Once confirmed it is tiering, change it back to default.

- Check space usage
df -aggregates -composite -aggregate aggrname
volume show-footprint
aggregate show-space
```

#### Other Misc Notes
```
- When deleting and detaching aggregate, can use the below command to verify aggr has been fully removed.
  storage aggregate object-store show-freeing-status
- When moving or deleting volumes to free up the aggregate, do not delete the aggregate until the scanners have completed.  Even though the aggregate may not contain any volumes, WAFL is still running scanners in the background.
node run -node nodename -command "priv set diag; wafl composite stats counter show aggrsrc" -> verify there are zero objects in bin1; bin0 is local tier and bin1 is capacity tier.
node run -node nodename -command "priv set diag; wafl scan status -A aggrsrc" -> check for cloud bin garbage collection scan message
- Use aws cli to list and delete objects in a bucket
aws --endpoint-url https://ipaddr/path_to_endpoint s3 ls s3://bucketname recursive
aws --endpoint-url https://ipaddr/path_to_endpoint s3 ls s3://bucketname recursive
```
