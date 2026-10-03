# Module 02 — Disks for Servers: EBS & EFS

← [All tutorials](../README.md) · **Storage tutorial**, module 2 of 3

EC2 instances need disks. Sometimes several servers also need to share the same files. AWS covers the first with **EBS** (a disk for one server) and the second with **EFS** (a shared network file system).

---

## 1. Amazon EBS: a disk for one server

An **EBS volume** is a network-attached disk. It appears in the operating system as a normal block device (`/dev/nvme1n1`), you format and mount it like any disk, and **it keeps its data independently of the instance**. Stop the instance, or even terminate it, and the volume can survive.

The rules that shape how you use it:
- **One AZ.** A volume lives in one Availability Zone and **attaches only to instances in that AZ**. To move data to another AZ or Region, you go through a **snapshot**.
- **One instance at a time.** (A special io2 *multi-attach* mode exists, but only for clustered software built for shared disks.)
- **The root volume** (the one the OS boots from) is created from the AMI at launch. By default it's **deleted when the instance is terminated** (`DeleteOnTermination`). Extra **data volumes** are kept by default.

### Volume types

| Type | What it's for | Performance |
|---|---|---|
| **gp3** (general purpose SSD) | **The default for almost everything:** boot disks, most databases on EC2 | 3,000 IOPS and 125 MB/s included, adjustable independently of size |
| **io2 Block Express** | Critical, latency-sensitive databases | Very high IOPS, lowest latency, highest durability |
| **st1** (throughput HDD) | Big sequential reads and writes: logs, data processing | High throughput, poor random I/O |
| **sc1** (cold HDD) | Rarely accessed data | Cheapest |

(IOPS = input/output operations per second: how many small reads or writes the disk handles.)

### Snapshots: backups and copies

A **snapshot** is a point-in-time backup of a volume, stored **regionally** (in S3, managed for you).
- Snapshots are **incremental**: each one stores only the blocks changed since the previous one. You can still restore any single snapshot as a full volume.
- **Restore into any AZ** of the Region, or **copy to another Region** for disaster recovery.
- Automate them with **AWS Backup** or the **Data Lifecycle Manager** (e.g. daily, keep 14).
- For a consistent snapshot of a busy database, pause writes or use the database's own backup mechanism.

### Other things to know

- **Encryption:** turn on *EBS encryption by default* for each Region in the EC2 settings. Every new volume and snapshot is then encrypted with KMS, with no performance cost.
- **Change a volume while it's in use** (*Elastic Volumes*): grow it, switch the type, or raise IOPS without stopping the instance. After growing, extend the partition and filesystem inside the OS (`sudo growpart /dev/nvme0n1 1 && sudo xfs_growfs -d /`).
- **Instance store** is something else: disks physically attached to the host on some instance types. It's very fast, but **its data is lost** when the instance stops or the host fails. Use it for caches and scratch space only.
- **Containers:** ECS can attach a fresh EBS volume to each task ([ECS 02](../ecs/02-task-definition-field-by-field.md)).

---

## 2. Amazon EFS: a file system shared by many servers

**EFS** is a managed **NFS** file system (NFS is the standard Linux network file sharing protocol). Many instances and containers, in **all AZs**, mount the same file system at the same time and see the same files. It **grows and shrinks automatically**, with no size to provision, and you pay for what you store.

How it connects to your VPC (see the diagram in [Module 01](01-storage-basics-and-s3.md)):
- You create a **mount target** in one subnet **per AZ**. That's a network interface with an IP, protected by a **security group** that must allow **NFS (TCP 2049)** from your servers' security group.
- The file system's DNS name (`fs-0123456789abcdef0.efs.eu-central-1.amazonaws.com`) resolves to the mount target **in the client's own AZ**.

Mounting it on Amazon Linux:

```bash
sudo dnf install -y amazon-efs-utils
sudo mkdir -p /mnt/shared
sudo mount -t efs -o tls fs-0123456789abcdef0:/ /mnt/shared        # tls = encrypted in transit
echo "fs-0123456789abcdef0:/ /mnt/shared efs _netdev,tls 0 0" | sudo tee -a /etc/fstab   # mount at every boot
```

Settings worth knowing:

| Setting | Options |
|---|---|
| Availability | **Regional** (data stored across several AZs, the default) or **One Zone** (cheaper, a single AZ) |
| Storage classes | **Standard** → **Infrequent Access** → **Archive**, moved automatically by a **lifecycle policy** based on last access |
| Throughput | **Elastic** (scales with your workload, the recommended default), or provisioned |
| Access points | Give each application its own root directory and Linux user, e.g. `/app1` as UID 1000 |
| Security | Security groups on the mount targets, optional IAM authorization, encryption at rest and in transit |

**Good for:** shared uploads and content, home directories, shared configuration, ML datasets, and containers that need a shared folder. **Not for:** databases or very latency-sensitive I/O (use EBS), or data accessed by API (use S3). For Windows file shares, Lustre (HPC), or NetApp ONTAP, look at the **Amazon FSx** family.

---

## 3. Try it: snapshot a volume and restore it in another AZ

No instance is needed. This shows the AZ rules in practice.

```bash
AZ1=$(aws ec2 describe-availability-zones --filters Name=zone-type,Values=availability-zone --query 'AvailabilityZones[0].ZoneName' --output text)
AZ2=$(aws ec2 describe-availability-zones --filters Name=zone-type,Values=availability-zone --query 'AvailabilityZones[1].ZoneName' --output text)

# A 4 GiB gp3 volume in AZ 1
VOL=$(aws ec2 create-volume --availability-zone $AZ1 --size 4 --volume-type gp3 --encrypted \
  --tag-specifications 'ResourceType=volume,Tags=[{Key=Name,Value=lab-vol}]' --query VolumeId --output text)
aws ec2 wait volume-available --volume-ids $VOL

# Change it while it exists: 8 GiB, more IOPS and throughput (no downtime needed; gp3 allows up to 500 IOPS per GiB)
aws ec2 modify-volume --volume-id $VOL --size 8 --iops 4000 --throughput 250 >/dev/null
aws ec2 describe-volumes-modifications --volume-ids $VOL --query 'VolumesModifications[0].[ModificationState,TargetSize,TargetIops]' --output text

# Snapshot it (a regional backup), then restore it as a new volume in AZ 2
SNAP=$(aws ec2 create-snapshot --volume-id $VOL --description "lab snapshot" --query SnapshotId --output text)
aws ec2 wait snapshot-completed --snapshot-ids $SNAP
VOL2=$(aws ec2 create-volume --availability-zone $AZ2 --snapshot-id $SNAP --volume-type gp3 --query VolumeId --output text)
aws ec2 wait volume-available --volume-ids $VOL2
aws ec2 describe-volumes --volume-ids $VOL $VOL2 --query 'Volumes[].[VolumeId,AvailabilityZone,Size,SnapshotId]' --output table
#   the same data, now in two AZs: that's how you move a disk between AZs

# Clean up
aws ec2 delete-volume --volume-id $VOL2
aws ec2 delete-volume --volume-id $VOL
aws ec2 delete-snapshot --snapshot-id $SNAP
```

---

## Check yourself

<details><summary>Can an EBS volume in AZ a be attached to an instance in AZ b?</summary>No. Snapshot it and create a new volume from the snapshot in AZ b.</details>
<details><summary>Ten containers in two AZs need to read and write the same files. EBS or EFS?</summary>EFS. An EBS volume is one instance in one AZ.</details>
<details><summary>What must the security group on an EFS mount target allow?</summary>TCP 2049 (NFS) from the clients' security group.</details>
<details><summary>Which EBS type do you start with?</summary>gp3. Move to io2 only for demanding databases.</details>

---
**Previous:** [Module 01](01-storage-basics-and-s3.md) · **Next:** [Module 03 — Databases & Caches](03-databases-and-caches.md)
