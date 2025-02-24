## VMware on NetApp Best Practices
* Use VSC Virtual Storage Console to manage vSphere with ONTAP.  It provides:
  - A single integrated admin interface
  - Ease of administration
  - Policy based management
  - Standard proceures and less mistakes
  - Best practice settings
  - Monitoring of best practice settings
  - VVol datastore provisioning
* Use NetApp Interoperability Matrix to confirm compatibility between software versions.
* Install NFS VAAI Plug-in on ESXi hosts if using NFS.  VAAI is auto enabled for SAN supported systems.  VAAI offloads tasks from the hosts to the storage system.
* Install the Guest OS scripts or VMware Tools
* Enable VASA Provider
* Register OnCommand API Services with VASA Provider if using Vvols
* Additional benefits of using VVols over traditional datastores
  - Vvols provide virtual disk granular storage
  - They provide enhanced features, such as VMware managed NetApp snapshot copies
  - They support Adaptive QoS
* Storage Protocol Choice - NetApp leans towards using NFS
  - You can use NFS or any SAN protocol with VMFS for your vSphere Datastores
  - NetApp testing has shown little performance difference between protocols
  - Considerations:
    * Current storage infrastructure and staff skills
    * Cost of any upgrades, i.e. cost to switch over from NAS to SAN
* Storage Protocol Choice - NFS Benefits
  - ONTAP has awareness of VMDK files.  You don't get that if using SAN however can get this awareness if using vvols in a SAN environment.
  - Provisioning can be easier on an NFS NIC than an HBA (easier to configure, less likely to require firmware management)
  - Thin provisioning, deduplication and cloning return savings immediately, which can make capacity management easier
  - Volumes can be easily shrunk or grown.  With SAN volumes can only be grown.
* Larger datastores can benefit storage efficiency (aggregate level deduplication on AFF alleviates the impact of this)
* Larger datastores also provide easier management because there are less datastores to manage
* Consider using at leaset four datastores (or volumes if using vvols) to store your VMs on a single ONTAP controller to get maximum performance from the hardware resources.
* This approach also allows you to establish datastores with different recovery policies (some backed up or replicated more frequently than others)
* Datastore recommended size is 4TB to 8TB.  Good balance point for performance, ease of management, and data protection.  This is big enough to hold lots of VMs, but small enough to recover quickly in case of a disaster.
* Enable autosize to automatically grow and shrink the volume as used space changes (VSC enables it by default)
* For workloads which require a large datastore, NFS datastores can be up to 100TB (300TB in newer releases) in size, VMFS datastores can be up 64TB in size.  If SAN is used, the ONTAP maximum LUN size depends on version you are running. 9.7 and lower is 16TB, 9.8 and later is 16TB for non-ASA platforms and 128TB for ASA platforms, 9.12.1P2 and later is 128TB for all platforms.
* For SAN, if using 16TB limit, it is not recommended to span a datastore across multiple LUNs in order to get a larger datastore.  There might be some performance benefit for high-I/O workloads, but this benefit is offset by added management complexity and availability risks.  NetApp generall recommends using a single, large LUN for each datastore and only span if there is special need.
* For AFF, FlashPool, secondary workload configurations, use the default aggregate and volume storage efficiency settings.  For HDD systems that require performance, disable the inline storage efficiency settings. 
*  Thin provisioning is the default for and recommended for NFS volumes.  Thin provisioned VMDKs are also generally recommended, depending on the application.
*  If using SAN, thin provisioned LUNs are recommended in order to see the benefits of deduplication.  LUNs are recommended to be created in thin provisioned volumes which are 2 times the size of the LUN.
*  Use a dedicated network for storage traffic.  ESXi hosts and SVM LIFs should be on the same IP subnet.  Make sure there are redundant paths and at least 1 LIF per node per SVM.
*  NFS Best Practices:
   - Use FlexVol volumes for NFS datastores (not qtrees or FlexGroups)
   - All ESXi hosts mounting an NFS datastore must use the same protocol (NFSv3 or v4.1).  Mixing could cause data corruption.  Use Host Profiles in VMWare to ensure this.
* VSC backup best practices
  - Backup your vSphere environment (you can use SnapCenter)
  - Redundancy should be provided for the VSC virtual machine through the standard vSphere High Availability or Fault Tolerance features
  - Information in VSC and the VASA Provider can change frequently - take snapshots at least every hour
  - Retain snapshots for enough time to discover and resolve andy problems
* Use a guest owned file system or RDM (Raw Device Mapping) if recommended for the guest application
* if enabling encryption, use NetApp (not vSphere) encryption
* Storage Distributed Resource Scheduler (DRS - moves VMs around to balance the load across the available datastores) should be set to manual if enabled.  Moves can lose efficiency benefits and lock space in snapshot copies.
* For SAN, use consistency groups if applications require high data integrity across multiple VMs
