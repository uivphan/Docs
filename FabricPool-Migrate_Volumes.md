

## Migrate data from StorageGrid to stand alone aggregate
```
  - Requirements
    * Destination aggregate not attached to capacity tier
    * Destination aggregate needs to have enough space available

  - Possible warning messages and their meanings
    ...volume will lose cross-volume deduplication because the destination is a newly-moved volume -> if the volume being moved has other duplicate data in other volumes in same aggregate, then the data on the newly moved volume will be undeduped and will deduped again on the destination aggregate if their are other volumes on the destination aggregate with duplicate data.

  - Check storage efficiency settings
    volume efficiency show -fields cross-volume-inline-dedupe
    volume efficiency config -volume volumename
    aggr efficiency show -aggregate aggrsrc
    aggr efficiency show -aggregate aggrdest

  - Steps to move flexvol to non-FabricPool attached aggregate.  You can change the tiering policy settings during the volume move.
    set -confirmations on
    volume show -vserver svmname -volume volumename -fields size,used
    volume move target-aggr show -vserver svmname -volume volumename
    volume move start -vserver svmname -volume volumename -destination-aggregate aggrdest -perform-validation-only true
    volume move start -vserver svmname -volume volumename -destination-aggregate aggrdest -tiering-policy none
    volume move show

  - Steps to move FlexGroup constituent to non-FabricPool attached aggregate.  Supported with 9.5 and higher.
    * Identify the constituents that consume the most space in the capacity tier
      volume show -vserver svmname -volume volumename__xx -fields size,used
      volume show-footprint -vserver svmname -volume volumename__xx -fields volume-blocks-footprint-bin1
    * Identify aggregates with free space
      volume move target-aggr show -vserver svmname -allow-mixed-aggr-types true -volume volumename__xx
    * Move the constituent volume 
      set -confirmations on
      volume move start -vserver svmname -volume volumename__xx -destination-aggregate aggrdest -allow-mixed-aggr-types true -perform-validation-only true
      volume move start -vserver svmname -volume volumename__xx -destination-aggregate aggrdest -allow-mixed-aggr-types true
      volume move show
      volume show-footprint -volume volumename__xx -fields volume-blocks-footprint-bin1
      volume show -volume volumename__xx -fields tiering-policy,aggregate
```

## Migrate data from StorageGrid to S3 using volume move
```
    - Choose this method if you have enough space in existing stand alone aggregates, can add more disks or systems to the cluster, can't tolerate tiering performance degradation, want flexibility withe moving volumes.
    - Requirements
    * Destination aggregates attached to the S3 object store
    * Aggregates need to have enough space for the volumes being moved
    * Configure the S3 bucket to be a little bigger in size than the StorageGrid bucket
    * When moving FlexGroup volumes, try to keep the constituents layout the same as the source aggregates

  - Possible warning messages and their meanings
    ...volume will lose cross-volume deduplication because the destination is a newly-moved volume - if the volume being moved has other duplicate data in other volumes in same aggregate, then the data on the newly moved volume will be undeduped and will deduped again on the destination aggregate if their are other volumes on the destination aggregate with duplicate data.

  - Check storage efficiency settings, volume tiering policies, aggregate tiering threshold
    volume efficiency show -fields cross-volume-inline-dedupe
    volume efficiency config -volume volumename
    aggr efficiency show -aggregate aggrsrc
    aggr efficiency show -aggregate aggrdest
    aggr object-store show -fields tiering-fullness-threshold
    volume show -volume volumename -fields tiering-policy,tiering-minimum-cooling-days

  - Steps to move flexvol.  You can change the tiering policy settings during the volume move.
    * Move the volumes
      set -confirmations on
      volume move target-aggr show -vserver svmname -volume volumename
      volume move start -vserver svmname -volume volumename -destination-aggregate aggrdest -perform-validation-only true
      volume move start -vserver svmname -volume volumename -destination-aggregate aggrdest
      volume move show
    * Delete the empty aggregates that are attached to StorageGrid, recreate them and attach them to the S3 object store.  Then move the rest of the volumes until there is no more data on the StorageGrid.
      storage aggregate object-store show-freeing-status
      aggr offline -aggregate aggrsrc
      aggr delete -aggregate aggrsrc
      aggr create -aggregate aggrsrc -node nodename -maxraidsize raidsize -diskcount numdisks -raidtype raid_dp
      node run -node nodename snap sched -A aggrsrc 0
      node run -node nodename snap reserve -A aggrsrc 0
      aggr efficiency show -aggregate aggrsrc
      aggregate object-store attach -aggregate aggr_noflexgroup -object-store-name s3svm-s3 - No FlexGroup  
      aggregate object-store attach -aggregate aggr_flexgroup -object-store-name s3svm-s3 -allow-flexgroup true - FlexGroup
      aggregate object-store show -fields tiering-fullness-threshold
      aggregate object-store modify -tiering-fullness-threshold 50 (50 is the default)      
    * Delete the StorageGrid bucket after object count shows zero

  - Steps to move FlexGroup.  Will be moving the constituent volumes.
      volume show-footprint -vserver svmname -volume volumename__xx 
    * Identify aggregates with free space
      volume move target-aggr show -vserver svmname -allow-mixed-aggr-types true -volume volumename__xx
    * Move the constituent volumes
      set -confirmations on
      volume move start -vserver svmname -volume volumename__xx -destination-aggregate aggrdest -allow-mixed-aggr-types true -perform-validation-only true
      volume move start -vserver svmname -volume volumename__xx -destination-aggregate aggrdest -allow-mixed-aggr-types true
      volume move show
      volume show-footprint -volume volumename__xx -fields volume-blocks-footprint-bin1
      volume show -volume volumename__xx -fields tiering-policy,aggregate
    * Delete the empty aggregates that are attached to StorageGrid, recreate them and attach them to the S3 object store.  Then move the rest of the constituents until there is no more data on the StorageGrid.
      storage aggregate object-store show-freeing-status
      aggr offline -aggregate aggrsrc
      aggr delete -aggregate aggrsrc
      aggr create -aggregate aggrsrc -node nodename -maxraidsize raidsize -diskcount numdisks -raidtype raid_dp
      node run -node nodename snap sched -A aggrsrc 0
      node run -node nodename snap reserve -A aggrsrc 0
      aggr efficiency show -aggregate aggrsrc
      aggregate object-store attach -aggregate aggr_noflexgroup -object-store-name s3svm-s3 - No FlexGroup  
      aggregate object-store attach -aggregate aggr_flexgroup -object-store-name s3svm-s3 -allow-flexgroup true - FlexGroup
      aggregate object-store show -fields tiering-fullness-threshold
      aggregate object-store modify -tiering-fullness-threshold 50 (50 is the default)    
    * Delete the StorageGrid bucket after object count shows zero
```

## Migrate data from StorageGrid to S3 using FabricPool Mirroring
```
  - Choose this method if you don't have enough space in existing stand alone aggregates, don't want to add more disks or systems to the cluster, can tolerate tiering performance degradation
  - Notes
    * Supported on 9.7 and higher.
    * When using FabricPool Mirror, data is mirrored across two buckets. During bucket synchronization, data must be read from the pre-existing primary bucket and written to the secondary bucket. This synchronization is necessary to achieve a mirrored state between the two buckets. When both buckets are in a mirrored state, newly tiered data is synchronously tiered to both buckets. Because data is being tiered to two buckets synchronously, the effective throughput is half of standard single-bucket tiering. Under normal circumstances, all GET operations take place from the primary bucket. Only if connectivity is interrupted to the primary bucket will GET operations take place from the secondary bucket. If connectivity is lost to either bucket, tiering is temporarily suspended until connectivity is established.
    * Mirroring is useful to maintain availability while performing maintenance on your object store, or for transitioning from one cloud tier to another for a given aggregate.

  - Requirements
    * Aggregate S3 object store config done
    * Configure the S3 bucket to be a little bigger in size than the StorageGrid bucket

  - Check storage efficiency settings, volume tiering policies, aggregate tiering threshold
    volume efficiency show -fields cross-volume-inline-dedupe
    volume efficiency config -volume volumename
    aggr efficiency show -aggregate aggrsrc
    aggr efficiency show -aggregate aggrdest
    aggr object-store show -fields tiering-fullness-threshold
    volume show -volume volumename -fields tiering-policy,tiering-minimum-cooling-days

  - Attach the S3 object store to the existing aggregate
    storage aggregate object-store mirror -aggregate aggrsrc -object-store-name s3objectstorename
    storage aggregate object-store show-resync-status -aggregate aggrsrc
    storage aggregate object store show
    storage aggregate object-store show -fields mirror-type,is-mirror-degraded,pending-op
    storage aggregate object-store modify -aggregate aggrsrc -object-store-name s3objectstorename -mirror-type primary
    storage aggregate object-store show -fields mirror-type,is-mirror-degraded,pending-op
    storage aggregate object-store unmirror -aggregate aggrsrc
    storage aggregate object-store show -fields mirror-type,is-mirror-degraded,pending-op
  * Delete the StorageGrid bucket after object count shows zero
```

## Migrate data to another aggregate attached to the same bucket
```
  - Notes
    * ONTAP supports both optimized (data in the capacity tier does not need to be copied over) and non-optimized (all data will be copied from the cloud tier) volume moves.  You can specify the -is-capacity-tier-optimized true pararmet if you want it to be optimized volume move.  If not specified, optimized volume move is attempted, and if it is not supported, non-optimized volume move will be performed automatically.  Can check for type in the mgwd log files.
    * Aggregates need to have enough space for the volumes being moved

  - Check storage efficiency settings, volume tiering policies, aggregate tiering threshold
    volume efficiency show -fields cross-volume-inline-dedupe
    volume efficiency config -volume volumename
    aggr efficiency show -aggregate aggrsrc
    aggr efficiency show -aggregate aggrdest
    aggr object-store show -fields tiering-fullness-threshold
    volume show -volume volumename -fields tiering-policy,tiering-minimum-cooling-days

  - Move the volumes
    set -confirmations on
    volume move target-aggr show -vserver svmname -volume volumename
    volume move start -vserver svmname -volume volumename -destination-aggregate aggrdest -perform-validation-only true
    volume move start -vserver svmname -volume volumename -destination-aggregate aggrdest
    volume move show
  * Delete the empty aggregates that is attached to the bucket.  It's possible zombie entries are left behind to track objects which require cleaning up.  This can occur because it was an optimized move. Once the objects are freed, the zombie entries will be removed.  The task to perform the clean up is a background task and depending on the load on the controller will take time to occur.
    node run -node nodename -command "priv set diag; wafl composite stats counter show aggrsrc" -> verify there are zero objects in bin1; bin0 is local tier and bin1 is capacity tier.
    node run -node nodename -command "priv set diag; wafl scan status -A aggrsrc" -> check for cloud bin garbage collection scan message
    aggr offline -aggregate aggrsrc
    aggr delete -aggregate aggrsrc
```
