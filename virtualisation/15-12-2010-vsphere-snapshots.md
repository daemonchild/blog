# vSphere Snapshots
**15th December 2010**


I often talk to people who don’t understand what VMware snapshots are for and how they work. Common bad practices include using them as a backup technique, or applying them for the long term when applying patches. I hope that the following article might be useful to explain how snapshots work in vSphere. I’ll talk about the hows and whys in this article.

## Where Snapshots are Useful
Snapshots are properly used when you need to do the following:

* Take a backup of a live running virtual machine
* Apply a patch or update to a virtual machine
* Perform any risky configuration to a virtual machine
* Make the disk file of a virtual machine read only

The implication in the above tasks is that they are a temporary state. A backup runs for as long as it takes to copy the virtual machine disk file to another location and then it’s over. A patch is applied and it either works fine or trashes the system; you’ll know within a short period of testing which is the case. All of these are temporary actions, and the snapshot that supports them should be a temporary item too.
It is vital that you understand that snapshots should never be used as a long term backup strategy for a VM. VMware snapshots are not entirely like storage based snapshots as I hope you will see from the rest of this article.

## How vSphere Snapshots Work

Before we talk about some more about snapshots, let’s get a few things sorted. We’ll quickly define the normal state of affairs for a virtual machine. In most cases, a virtual machine stores its disk blocks inside a file. This is known as a VMDK file, standing for Virtual Machine DisK.

There are actually two files for every virtual disk. One is tiny and defines the drive geometry – that is, the number of cylinders, sectors and heads – so that the VM BIOS knows how to address the disk. The second file holds the actual data. I find it convenient to think of this as a matrix of disk blocks, as shown in the following diagram. (As always, click the picture for a zoomed version.)

![Image 1](https://github.com/daemonchild/blog/images/15122010-vsphere-snapshots-1.png "Image 1")


The VM is depicted as reading and writing from this bunch of disk blocks. This notation is important for our discussion, because it is the target of this I/O that changes when we add a snapshot to a VM. In this discussion, we’re talking about virtual machine disk snapshots rather than memory snapshots by the way. The concept is similar, but it’s the disk snapshots that tend to get admins into trouble.

So lets, add a snapshot to our virtual machine configuration. In vSphere, what happens is that a log file is created. This log file is rather like a ‘redo’ log for a transactional database. The initial size of this log file is 16MB and it will grow in 16MB chunks. Notice that the snapshot log file is not the same size as the VMDK. It is definitely not a copy or clone of the entire VMDK file – which might be the implication for some people in the name ‘snapshot’. An important point is that the snapshot log file will keep growing until you tell it to go away. This means that, if it is forgotten about for a long time, it will fill you VMFS datastore. This can be tricky to recover from and could effect a whole bunch of VMs, especially if you have thin provisioned any of them. This scenario is, frankly, a real nightmare. Think about it.

The write I/O of the VM is redirected into this snapshot file.


![Image 2](https://github.com/daemonchild/blog/images/15122010-vsphere-snapshots-2.png "Image 2")



This means that the VMDK file is effectively rendered read only. It has been freed up to be safely copied or moved such as in a cloning or storage vMotion operation, or during a backup. All disk writes will be recorded into the snapshot log file in a linear list. We’ll talk about what we can do with this data shortly.

I hope that you can see that the read I/O for the virtual machine could come from two places. The snapshot file contains the most up to date version of any given disk block that we have written since the snapshot was added. The VMDK file contains all other disk blocks. Sometimes, the vmkernel will have to consult both sources to collect the data that a VM is requesting. This does not help VM disk performance.

In addition, every time that the snapshot file needs to grow by another 16MB chunk, the ESX server in control of it needs to lock the entire LUN (I need to check whether this is still true in vSphere 4.1 because of some changes to disk locking that I heard about). This would affect all the other virtual machines on the LUN too – only for a tiny fraction of a second, but add this up and it becomes a sizeable effect.

Ok. So we’re finished with our snapshot, what choices do we have? There are essentially three:

* Merge the data with he VMDK
* Destroy the snapshot log file
* Add another snapshot

**Merge**

Assuming that the snapshot is being used to allow a backup to take place the snapshot data will be kept. We want the changes that took place during the backup to be merged with the data in the VMDK. The vmkernel needs to trawl through the log file and overwrite the data in the VMDK file with the most up to date copy from the snapshot log file. The diagram aims to show this happening.


![Image 3](https://github.com/daemonchild/blog/images/15122010-vsphere-snapshots-3.png "Image 3")



Sometimes, the snapshot file gets quite large during the time that it was in service – perhaps the VM was quite busy, or maybe the snapshot was active for some time. In this case, it is necessary for the vmkernel to automatically snapshot the virtual machine again so that the original log file can be rendered read only for merging. You’ll notice this in the diagram called a Consolidated Helper Snapshot (CHS). Actually, sometimes, the original snapshot gets so large that the CHS also gets pretty big. Guess what? Thats right, the vmkernel will add a second CHS and a third and so on like Russian dolls until the last one is small enough to be dealt with in a very short time. Ordinarily, you should even see more than one CHS; seeing more than one on your VM should serve as a warning to you that your snapshots are growing large.

**Destroy**

Another possibility is that we don’t want the data that has accumulated inside the log file. Perhaps this is the result of a failed upgrade or patch. The patch changes will be inside the log file, so we could ask vSphere to destroy this data and return to the original base VMDK file.


![Image 4](https://github.com/daemonchild/blog/images/15122010-vsphere-snapshots-4.png "Image 4")



In this case all the data inside the log file will lost. I/O reverts to the base disk. To support this, the virtual machine will have to be rebooted and will come up in what is known as a ‘crash-consistent’ state. Why would this be the case? The snapshot was taken at a point in time. Whatever the disk state at that time was essentially frozen – hence the term snapshot. To return to this point, we need the current state of the memory to be consistent with that point in time. So the machine is rebooted to reload the OS based on the previous disk state. It’s no good having the contents of a SQL database in memory if the temporary files that support it have been removed from the disk.

I want to reiterate something here: all the data inside the log file will lost. In my example, we’re destroying the data because of a failed patch. If the VM was quiet when the failed patch was applied, then we’re good to go. By quiet, I mean that there was no user interaction with it; there were no changes made by users or other important processes. However, if the VM was active then we have a small problem. The snapshot data file will include changes made by the failed patch and also user data. The quandary is now whether to delete the data file to remove all evidence of the patch but also knowing that you will lose all user changes since the snapshot..

If some of the snapshot data is user data then you will lose important changes to the virtual machine. There is no way that the vmkernel can differentiate inside the snapshot. In fact, the principle of isolation means that the vmkernel is not allowed to know what your disk writes actually contain.

The only way to ensure that this is not going to happen is to stop users accessing the VM while you apply patches. There is no magic bullet here, but at least it’s easier that in the physical world. (Hint: drop the VM into an isolated network vSwitch so that you don’t have to mess about with stopping services.)

**Add Another Snapshot**

Another possible choice is to add a second snapshot on top of the first. A good example of when to use this might be that you are applying a series of upgrades or patches that have interdependencies. If the first goes well, you might choose to attempt the next stage. So we might snapshot at each step, meaning that we can roll back just the last one or two step should something go bad, rather than starting over back at the base disk.


![Image 5](https://github.com/daemonchild/blog/images/15122010-vsphere-snapshots-5.png "Image 5")



**Recap: Reasons for not using snapshots long term**

I hope that you’ve been able to deduce the following reasons for not using snapshots as anything more than a useful tool to temporarily save the state of a VMDK file. Here are the reasons not to leave a snapshot applied for a long time:

* Performance decrease to VM caused by multiple I/O sources
* Performance decrease to LUN caused by multiple lock/unlock operations
* Risk of user data loss if you need to destroy the snapshot
* Snapshots keep growing until they fill your VMFS datastore

