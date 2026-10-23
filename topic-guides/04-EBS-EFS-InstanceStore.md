# Section 4: EBS, EFS & Instance Store

## The idea

Your EC2 instance needs somewhere to put its files. AWS gives you three very different kinds of "somewhere," and the exam loves making you pick between them.

Here's the analogy: think of your instance as a desk in an office.

- **EBS (Elastic Block Store)** is your **personal external hard drive** plugged into your desk. It's actually a network-attached drive (the data lives on separate storage servers, not inside your computer), but it feels local. Unplug it from one desk, plug it into another. It's *yours alone*.
- **EFS (Elastic File System)** is the **shared network folder** the whole office mounts. Everyone sees the same files at the same time.
- **Instance Store** is the **notepad glued to the desk itself**. Blazing fast to scribble on — but when you give up the desk, the notepad goes in the shredder.

Three storage types, three personalities: **private and persistent (EBS), shared (EFS), fast but disposable (Instance Store).** Let's take them in turn.

## EBS: your network-attached hard drive

Rules of the game — each one is exam-tested:

- An EBS volume lives in **exactly one Availability Zone**. To move it to another AZ, you snapshot it and restore the snapshot there.
- It attaches to **one instance at a time** — with one exception: **io1/io2 Multi-Attach**, which lets up to **16 Nitro-based instances in the same AZ** attach the same volume (for cluster-aware software only, not ordinary filesystems). Standard file systems such as XFS/EXT4 are not safe for concurrent multi-host writes without a cluster-aware layer.
- It **persists independently** of the instance — the drive outlives the desk...
- ...**except** for the deletion flags: the **root volume** is created automatically and has **DeleteOnTermination = true by default** (terminate the instance, lose the root disk). **Extra attached volumes default to false** — they survive. So *"the root volume must survive termination"* → **turn DeleteOnTermination off** for the root volume.
- A freshly attached extra volume is a **raw disk**: you must **format it, mount it, and add it to /etc/fstab** yourself, or it vanishes from the mount table on reboot.

### The one axis that decides everything: IOPS vs Throughput

Every EBS question secretly asks: is this workload about **many small random operations** or **big sequential streams**?

- **IOPS** (input/output operations per second) = **lots of small, random reads/writes**. That's what **databases** do — hop around the disk grabbing tiny rows.
- **Throughput** (MB/s) = **large, sequential** reads/writes. That's **big data, log processing, ETL, streaming** — chewing through giant files front to back.

The full decision tree (volume and EC2 instance limits both apply; an instance can bottleneck an otherwise fast volume):

```
Random I/O? Database? Boot volume?
        │
        ▼ yes                          no ▼ (big sequential data)
      SSD                                HDD  ← cannot be a boot volume!
        │                                 │
  ≤ 80,000 IOPS?                   accessed frequently?
   │          │                     │            │
   ▼ yes      ▼ no                  ▼ yes        ▼ no (cold)
  gp3      io1/io2                 st1          sc1
 (default) (provisioned IOPS;   (throughput-  (cheapest
            io2 Block Express    optimized     EBS there is)
            → 256,000 IOPS,      HDD)
            sub-ms latency,
            Multi-Attach)
```

**The old 16,000 IOPS threshold is outdated.** Current gp3 volumes support up to **80,000 IOPS** (at 160 GiB or larger) and **2,000 MiB/s**; volume and instance limits both matter. Need more than gp3 supports, or a provisioned-IOPS profile / higher durability? Consider io1/io2. **io2 Block Express** supports up to **256,000 IOPS on Nitro instances** and average sub-500-microsecond latency for 16 KiB I/O; non-Nitro limits are lower. Since April 30, 2025, all new and existing io2 volumes are Block Express, with performance depending on the attached instance. **HDD volumes (st1/sc1) can never be boot volumes.**

### gp2 vs gp3: the free-money question

| | gp2 (old) | gp3 (new) |
|---|---|---|
| IOPS | **3 IOPS per GiB** (minimum 100, maximum 16,000); smaller volumes can burst to 3,000 | **baseline 3,000 IOPS**; provision up to 80,000, limited by size (500 IOPS/GiB) |
| Throughput | size-dependent, up to 250 MiB/s | **baseline 125 MiB/s; provision up to 2,000 MiB/s** |
| Scaling performance | must grow the volume to grow IOPS | **provision IOPS and throughput independently** |
| Price | baseline | **about 20% lower storage price in many Regions (compare current regional pricing)** |

On gp2, wanting more IOPS meant buying storage you didn't need. gp3 broke that chain. So the moment a question mentions **gp2**, first compare its current size/performance needs with gp3; for many workloads, gp3 is a straightforward, lower-cost upgrade. Elastic Volumes can modify type/performance online for supported volumes. And *"needs more IOPS without paying for more storage"* → **gp3** (independent provisioning). It's the exam's favorite free lunch.

### Snapshots: the time machine

- Snapshots are **incremental backups stored in S3** — only changed blocks after the first one, so they're cheap.
- You can **copy a snapshot to another region** — that's the answer to **cross-region disaster recovery** ("back up EBS data to another region" → snapshot, copy it over).
- **Snapshot Archive tier**: up to **75% lower storage cost** for snapshots kept at least 90 days, but restoring can take **up to 72 hours**. Archiving converts an incremental snapshot to a full snapshot. Keyword: "rarely restored, minimize snapshot cost."
- **Recycle Bin**: retention rules that catch accidentally deleted snapshots so you can recover them.
- **Fast Snapshot Restore (FSR)**: normally a volume restored from a snapshot is lazily loaded (first read of each block is slow). FSR pre-warms it for **full performance instantly — but it's expensive**. Keyword: "no latency on first access after restore."
- **Data Lifecycle Manager (DLM)**: **automates** snapshot creation, retention, and deletion on a schedule. "Automate EBS backups" → DLM.

### Encryption: the snapshot-copy dance

Encryption is sticky in the happy direction: **an encrypted volume produces encrypted snapshots, and encrypted snapshots produce encrypted volumes.** Everything downstream inherits it, with essentially no performance cost.

But **you cannot flip encryption on for an existing unencrypted volume in place**. Create a snapshot, then create a new encrypted volume directly from that snapshot (or copy the snapshot with encryption enabled first if you need an encrypted snapshot, cross-Region copy, or key change):

```

unencrypted volume → snapshot it → create a new encrypted volume from
that snapshot → swap it in

Optional: copy the snapshot with encryption enabled first if the snapshot
itself must be encrypted, or you need a different KMS key/Region.
```

Any question about "encrypt an existing EBS volume" tests the **snapshot → new encrypted volume → swap** workflow. Copying the snapshot first is optional unless the snapshot itself must be encrypted or you need to change Region/key.

## EFS: the shared network folder

**EFS (Elastic File System)** is a managed **NFS** (Network File System — the classic Linux shared-folder protocol) file system. The personality traits:

- **Mounted by many instances at once.** Regional EFS stores data across multiple AZs; One Zone EFS stores data in a single AZ and is less resilient to AZ loss. This is the shared-drive use case.
- **Linux ONLY.** THE trap: **Windows instances cannot mount EFS. Windows shared storage → FSx for Windows File Server** (SMB protocol, Active Directory integration). Read the OS in the question before answering "EFS."
- **Elastic**: it grows/shrinks as files change; you pay for stored data, with per-file minimum billing and access/throughput charges in lower-cost tiers.
- Pricier per GB than EBS — but you buy zero unused space, and lifecycle tiers claw back cost.

Dials you can turn (exam wants recognition, not depth):

| Dial | Options | Pick when |
|---|---|---|
| Performance mode | **General Purpose** (default) / **Max I/O** (previous generation) | General Purpose is recommended for new file systems. Max I/O has higher per-operation latency, is unsupported with Elastic throughput, and is not available for One Zone. |
| Throughput mode | **Elastic** / Provisioned / Bursting | Elastic is recommended for spiky/unpredictable workloads; Provisioned for known sustained needs; Bursting scales with stored data |
| Lifecycle | Standard → **Infrequent Access (IA)** → **Archive** (Archive for Regional EFS) | Automatically tier cold files; IA/Archive reduce storage cost but have different access latency/charges |

**The signature use case:** a fleet of instances behind an Auto Scaling Group that must all see the same files — **user uploads, CMS content, WordPress wp-content**. "Multiple EC2 instances need shared access to the same files (Linux)" → **EFS**, every time.

## Instance Store: the notepad glued to the desk

**Instance Store** is physical disk **inside the host machine itself** — no network hop at all. Consequences:

- **Very fast local storage** with low latency and high throughput; exact performance depends on the instance type. It can outperform EBS for some workloads, but there is no single fastest choice for every workload.
- **EPHEMERAL** (temporary): data is **lost when the instance stops, hibernates, or terminates** — though it generally **survives a reboot**. It can also be lost if the underlying disk fails, so keep only reproducible or replicated data there.
- **Cannot be detached, cannot be snapshotted** through EBS mechanisms, size is fixed by the instance type.

Legit uses: **cache, buffer, scratch space, temp processing files**, or data that's **replicated at the application layer** anyway (Cassandra nodes replicate to each other, so losing one node's local disk is survivable).

**THE trap:** *"highest possible IOPS AND data must persist"* — instance store wins the IOPS contest but flunks persistence. The answer is **io2 (Block Express) EBS**. The word **"persist"** disqualifies instance store instantly.

## The five-storage lineup

| Storage | One-liner |
|---|---|
| **Instance Store** | Fastest, physical, ephemeral — cache/scratch/replicated data only |
| **EBS** | Network block device, one instance (mostly), one AZ, persistent — boot volumes and databases |
| **EFS** | Shared NFS folder for many Linux instances; Regional is multi-AZ, One Zone is single-AZ |
| **FSx** | Managed third-party file systems — **Windows (SMB/AD)**, or Lustre for HPC |
| **S3** | Object storage over HTTP — not a disk at all; apps talk to it via API, effectively unlimited |

## Question patterns

> *"Database needs 12,000 IOPS, cost-effective"* → **gp3** (within gp3 limits; provision IOPS independently of capacity)

> *"Critical database needs 50,000 IOPS"* → **gp3 or io2**, depending on latency, durability, workload, and instance limits; gp3 supports up to 80,000 IOPS, while io2 Block Express supports up to 256,000 on Nitro

> *"Sequential processing of large log files, throughput-focused, cost-effective"* → **st1** (big + sequential = HDD; frequent access = st1, cold archive = sc1 — and neither can boot)

> *"gp2 volume needs more IOPS but not more storage"* → **migrate to gp3** (gp2 chains IOPS to size; gp3 provisions them independently and is often about 20% lower in storage price (compare current regional pricing))

> *"Fleet of Linux instances in an ASG must share uploaded files"* → **EFS** (many mounters + multi-AZ + NFS = the shared folder)

> *"Windows instances need shared storage with Active Directory integration"* → **FSx for Windows File Server** (EFS is Linux-only — THE trap)

> *"Fast local storage for temporary scratch/cache data"* → **Instance Store** (if the instance type offers it and the data is disposable/reproducible)

> *"High IOPS storage whose data must persist after the instance stops"* → **io2 EBS, not instance store** ("persist" disqualifies the notepad)

> *"Data volume must survive instance termination"* → **DeleteOnTermination = false** (root defaults true, extra volumes default false — check which one the question means)

> *"Encrypt an existing unencrypted EBS volume"* → **snapshot → create a new encrypted volume from that snapshot → swap it in** (snapshot copy is optional if the snapshot itself must be encrypted or you need a different key/Region)

> *"Back up EBS volumes to another region for DR, automatically"* → **DLM-scheduled snapshots + cross-region snapshot copy** (snapshots live in S3 and copy across regions)

> *"Multiple instances must attach the same block volume in one AZ (cluster software)"* → **io1/io2 Multi-Attach** (up to 16 instances, same AZ — block storage, not a file system)

> *"Reduce cost of EFS for files untouched for months"* → **lifecycle policy to EFS IA/Archive** (same trick as S3 tiering, applied to the shared folder)

## Pocket card

| Keyword | Answer |
|---|---|
| random I/O / database / boot volume | SSD (gp3 or io1/io2) |
| large sequential / big data / throughput MB/s | HDD (st1/sc1) — never bootable |
| up to 80,000 IOPS (verify volume size and instance limits) | gp3 |
| > 80,000 IOPS, or provisioned IOPS requirements | io1/io2 |
| 256,000 IOPS / sub-millisecond | io2 Block Express |
| frequent sequential (logs, ETL) | st1 |
| coldest, cheapest EBS | sc1 |
| gp2 mentioned at all | migrate to gp3 (about 20% lower storage price in many Regions (compare current regional pricing)) |
| more IOPS without more storage | gp3 (independent provisioning) |
| multiple instances, one block volume, same AZ | io1/io2 Multi-Attach (max 16 Nitro instances; cluster-aware software) |
| root disk survives termination | DeleteOnTermination = false |
| new extra volume unusable | format + mount + fstab |
| move volume across AZ/region | snapshot → (copy) → restore |
| cheap rarely-restored snapshots | Snapshot Archive (up to 75% lower, 90-day minimum; restore up to 72 hr) |
| recover deleted snapshot | Recycle Bin |
| instant full performance from snapshot | Fast Snapshot Restore (expensive) |
| automate snapshot schedules | Data Lifecycle Manager |
| encrypt existing volume | snapshot → create encrypted volume from snapshot (copy optional) |
| shared files, many Linux instances | EFS Regional (multi-AZ) or One Zone (single AZ) |
| shared files, Windows / SMB / Active Directory | FSx for Windows (never EFS) |
| HPC parallel file system | FSx for Lustre |
| new EFS deployment | General Purpose performance mode (recommended); Max I/O is previous generation |
| EFS throughput mode | Elastic for spiky/unknown workload; Provisioned or Bursting for known patterns |
| cut EFS cost on cold files | lifecycle to IA / Archive |
| very fast local storage / temp / cache / scratch | Instance Store (ephemeral; performance varies by instance) |
| instance store + stop/terminate | data gone (reboot survives) |
| high IOPS + must persist | io2 EBS, not instance store |
| Cassandra/replicated data, local speed | Instance Store is OK |

You now know where an instance keeps its data — next up, S3 takes storage out of the instance entirely and into the world of objects, buckets, and eleven nines of durability.


## Current AWS references

- [EBS gp3 performance limits](https://docs.aws.amazon.com/ebs/latest/userguide/general-purpose.html)
- [io2 Block Express performance](https://docs.aws.amazon.com/ebs/latest/userguide/provisioned-iops.html)
- [EBS Multi-Attach requirements](https://docs.aws.amazon.com/ebs/latest/userguide/ebs-volumes-multi.html)
- [Encrypting existing EBS resources](https://docs.aws.amazon.com/ebs/latest/userguide/encryption-examples.html)
- [EBS snapshot archive](https://docs.aws.amazon.com/ebs/latest/userguide/snapshot-archive.html)
- [EFS Regional and One Zone file systems](https://docs.aws.amazon.com/efs/latest/ug/features.html)
- [EFS performance and throughput modes](https://docs.aws.amazon.com/efs/latest/ug/performance.html)
- [EFS lifecycle management](https://docs.aws.amazon.com/efs/latest/ug/lifecycle-management-efs.html)







