# QAT-Ceph Install and Test BKM

## Environment

CPU: Intel(R) Xeon(R) CPU E5-2699 v4 @ 2.20GHz

QAT Hardware: QAT Adaptor 8970

OS: Ubuntu 20.04

Kernel: 5.4.0-144-generic


## QAT Driver Install

QAT driver version: QAT.L.4.18.1-00001

```bash
export ICP_ROOT=/QAT
apt-get install -y libboost-dev yasm
mkdir -p ${ICP_ROOT}
cd ${ICP_ROOT} && wget https://downloadmirror.intel.com/738667/QAT.L.4.18.1-00001.tar.gz
tar -xvf QAT.L.4.18.1-00001.tar.gz -C ./
./configure
make
make clean
make install

systemctl restart qat_service
systemctl status qat_service
```

## QATzip Install

QATZip version: QATZip1.0.8_CPM20_LZ4_Beta_001

```bash
QATZIP_PATH=https://af01p-igk.devtools.intel.com/artifactory/platform_hero-igk-local/hero_features_assets/qathw/applications.qat.shims.qatzip.qatzip-QATZip1.0.8_CPM20_LZ4_Beta_001.tar.gz
apt-get install -y libxxhash-dev
apt-get install -y automake
export QZ_ROOT=/QATzip
mkdir -p ${QZ_ROOT} && cd ${QZ_ROOT} && wget ${QATZIP_PATH}
tar -xvf applications.qat.shims.qatzip.qatzip-QATZip1.0.8_CPM20_LZ4_Beta_001.tar.gz -C ${QZ_ROOT} && rm -rf applications.qat.shims.qatzip.qatzip-QATZip1.0.8_CPM20_LZ4_Beta_001.tar.gz
mv applications.qat.shims.qatzip.qatzip-QATZip1.0.8_CPM20_LZ4_Beta_001/* ./ && rm -r applications.qat.shims.qatzip.qatzip-QATZip1.0.8_CPM20_LZ4_Beta_001/
./autogen.sh && ./configure --with-ICP_ROOT=${ICP_ROOT} --enable-debug --enable-symbol
make clean
make all install
qzip --version
```

## Install Ceph

Ceph Version: 18.0.0 (dev)

```bash
CEPH_REPO=https://github.com/ceph/ceph.git
cd /opt && git clone ${CEPH_REPO}
git submodule update --init --recursive
./install-deps.sh
export ICP_ROOT=/QAT
./do_cmake.sh -DWITH_QATZIP=ON -DWITH_QAT=ON -DWITH_TESTS=OFF -DCMAKE_BUILD_TYPE=RelWithDebInfo
cd build
ninja -j$(nproc)
ninja install
```

Errors that may occur:

1. Cannot find librados.so.2

```bash
$ ceph -v
Traceback (most recent call last):
File "/usr/local/bin/ceph", line 140, in <module>
import rados
ImportError: librados.so.2: cannot open shared object file: No such file or directory
```

Need to specify the ceph path:

```bash
echo -e '/usr/local/lib/' | tee /etc/ld.so.conf.d/ceph.conf
ldconfig
```

2. Cannot find ceph_argparse

```bash
$ ceph -v
Traceback (most recent call last):
File "/usr/local/bin/ceph", line 146, in <module>
from ceph_argparse import \
ModuleNotFoundError: No module named 'ceph_argparse'
```

It's a python path issue, can solve this by copying ceph-related files to your python path:

```bash
sudo cp /usr/local/lib/python3/dist-packages/*.py /usr/local/lib/python3.8/dist-packages/
```


## Ceph Deploy

Tool: CeTune (https://github.com/hualongfeng/CeTune.git) 

### 1. Set Hostname

Edit /etc/hosts file on each ceph node, e.g. ceph-node-1, ceph-node-2, ceph-node-3
Check:
```bash
hostname -s
ping ceph-node-1
ping ceph-node-2
ping ceph-node-3
```

### 2. SSH Configuration

Ensure every ceph node can visit each other(including itself) via ssh without password:

```bash
ssh-keygen
ssh-copy-id root@{hostname}
```

### 3. SSD Partition

#### Our Goal
OSD number: 2 per node

NVMe:OSD=1:2

#### Expected Result:

```bash
nvme0n1                   259:0    0   1.8T  0 disk
├─nvme0n1p1               259:1    0    50G  0 part
├─nvme0n1p2               259:2    0    50G  0 part
├─nvme0n1p3               259:3    0    50G  0 part
└─nvme0n1p4               259:4    0    50G  0 part
nvme1n1                   259:5    0   1.8T  0 disk
├─nvme1n1p1               259:6    0     5G  0 part 
├─nvme1n1p2               259:7    0     5G  0 part 
├─nvme1n1p3               259:8    0   500G  0 part
└─nvme1n1p4               259:9    0   500G  0 part
```

#### Partition Process (Nvme1n1 as an example)

```bash
fdisk /dev/nvme1n1
Command (m for help): p
Disk nvme1n1: 1.5 TiB, 1600321314816 bytes, 3125627568 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0xe5bbdd11

Device     Boot    Start        End    Sectors  Size Id Type
nvme1n1p1           2048   10487807   10485760    5G 83 Linux
nvme1n1p2       10487808 3125627567 3115139760  1.5T 83 Linux

Command (m for help): d nvme1n1p2
Partition number (1,2, default 2):

Partition 2 has been deleted.

Command (m for help): p
Disk nvme1n1: 1.5 TiB, 1600321314816 bytes, 3125627568 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0xe5bbdd11

Device     Boot Start      End  Sectors Size Id Type
nvme1n1p1        2048 10487807 10485760   5G 83 Linux

Command (m for help): n
Partition type
   p   primary (1 primary, 0 extended, 3 free)
   e   extended (container for logical partitions)
Select (default p): p
Partition number (2-4, default 2):
First sector (10487808-3125627567, default 10487808):
Last sector, +sectors or +size{K,M,G,T,P} (10487808-3125627567, default 3125627567): +5G

Created a new partition 2 of type 'Linux' and of size 5 GiB.

Command (m for help): p
Disk nvme1n1: 1.5 TiB, 1600321314816 bytes, 3125627568 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0xe5bbdd11

Device     Boot    Start      End  Sectors Size Id Type
nvme1n1p1           2048 10487807 10485760   5G 83 Linux
nvme1n1p2       10487808 20973567 10485760   5G 83 Linux

Command (m for help): n
Partition type
   p   primary (2 primary, 0 extended, 2 free)
   e   extended (container for logical partitions)
Select (default p):

Using default response p.
Partition number (3,4, default 3):
First sector (20973568-3125627567, default 20973568):
Last sector, +sectors or +size{K,M,G,T,P} (20973568-3125627567, default 3125627567): +740G

Created a new partition 3 of type 'Linux' and of size 740 GiB.

Command (m for help): p
Disk nvme1n1: 1.5 TiB, 1600321314816 bytes, 3125627568 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0xe5bbdd11

Device     Boot    Start        End    Sectors  Size Id Type
nvme1n1p1           2048   10487807   10485760    5G 83 Linux
nvme1n1p2       10487808   20973567   10485760    5G 83 Linux
nvme1n1p3       20973568 1572866047 1551892480  740G 83 Linux

Command (m for help): n
Partition type
   p   primary (3 primary, 0 extended, 1 free)
   e   extended (container for logical partitions)
Select (default e): p

Selected partition 4
First sector (1572866048-3125627567, default 1572866048):
Last sector, +sectors or +size{K,M,G,T,P} (1572866048-3125627567, default 3125627567):

Created a new partition 4 of type 'Linux' and of size 740.4 GiB.

Command (m for help): p
Disk nvme1n1: 1.5 TiB, 1600321314816 bytes, 3125627568 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0xe5bbdd11

Device     Boot      Start        End    Sectors   Size Id Type
nvme1n1p1             2048   10487807   10485760     5G 83 Linux
nvme1n1p2         10487808   20973567   10485760     5G 83 Linux
nvme1n1p3         20973568 1572866047 1551892480   740G 83 Linux
nvme1n1p4       1572866048 3125627567 1552761520 740.4G 83 Linux

Command (m for help): w
The partition table has been altered.
Syncing disks.
```

#### Inform the Linux Kernel

```bash
partprobe /dev/nvme1n1
```

Info: Nvme0 is used to store OSD metadata(db_wal)(50G for each), Nvme1n1p1, Nvme1n1p2, Nvme2n1p1, Nvme2n1p2 are used to store basic info of OSD, and Nvme1n1p3, Nvme1n1p4, Nvme2n1p3, Nvme2n1p4 are used to store user data.

### 4. Deploy Ceph through CeTune

#### Clone CeTune

```bash
git clone https://github.com/hualongfeng/CeTune.git /opt/CeTune
```

#### Install CeTune Dependencies

```bash
apt-get install python
cd /opt/CeTune/deploy
python controller_dependencies_install.py  # for ceph controller node
cd /CeTune/deploy/
python worker_dependencies_install.py # for ceph worker nodes
```

#### Config CeTune all.conf

```bash
cd /opt/CeTune/conf
vi all.conf
```

Here is a demo for reference:
```bash
#all.conf
#============global============
distributed=0
#============benchmark============
tmp_dir=/mnt/opt
dest_dir=/mnt/result
cache_drop_level=3
monitoring_interval=1
perfcounter_time_precision_level=1
fio_capping=false
volume_size=61440
rbd_volume_count=90
disk_num_per_client=30,30,30
enable_zipf=false
fio_randrepeat=True
check_tuning=true
disable_tuning_check=false
perfcounter_data_type=osd,bluestore,rocksdb
distributed_data_process=true
rwmixread=100
collector=perfcounter
#============cluster============
clean_build=true
user=root
head=waikikibeach21
list_server=waikikibeach21,waikikibeach22,waikikibeach35
list_client=
list_mon=waikikibeach21,waikikibeach22,waikikibeach35
disk_format=osd:data:db_wal
waikikibeach21=/dev/nvme0n1p1:/dev/nvme0n1p3:/dev/nvme1n1p1,/dev/nvme0n1p2:/dev/nvme0n1p4:/dev/nvme1n1p2
waikikibeach22=/dev/nvme0n1p1:/dev/nvme0n1p3:/dev/nvme1n1p1,/dev/nvme0n1p2:/dev/nvme0n1p4:/dev/nvme1n1p2
waikikibeach35=/dev/nvme0n1p1:/dev/nvme0n1p3:/dev/nvme1n1p1,/dev/nvme0n1p2:/dev/nvme0n1p4:/dev/nvme1n1p2
enable_rgw=false
data_user=root
data_dir=/home/fhl/benchmark/result
monitoring_interval=1
cache_drop_level=3
#============ceph_hard_config============
auth_service_required=none
auth_cluster_required=none
auth_client_required=none
mon_data_avail_warn=1
mon_data=/var/lib/ceph/ceph.$id
osd_pool_default_pg_num=1024
osd_pool_default_pgp_num=1024
osd_pool_default_size=2
osd_pool_default_min_size = 2
osd scrub load threshold=0.001
osd scrub min interval=537438953472
osd scrub max interval=537438953472
osd deep scrub interval=537438953472
osd max scrubs=16
mon pg warn min per osd=0
mon pg warn max per osd=32768
osd_objectstore=bluestore
public_network=10.67.114.0/23 # modify according to your network
monitor_network=10.67.114.0/23
cluster_network=10.67.114.0/23
enable experimental unrecoverable data corrupting features=*
bluestore_bluefs=true
bluestore_block_create=false
bluestore_block_db_create=false
bluestore_block_wal_create=false
bluestore_block_wal_separate=false
mon_allow_pool_delete=true
ms_bind_msgr2=false
osd_pool_default_pg_autoscale_mode=off
ms_async_op_threads = 6
rgw_enable_quota_threads = 0
objecter_inflight_op_bytes = 1024_M
objecter_inflight_ops = 10_K
# rgw_numa_node = 0  #That rgw is on side numa is better
rbd_cache = false   #disable rbd_cache
bluestore_compression_mode=force
bluestore_compression_algorithm=zlib  # select zlib compression algorithm, this for all pool
qat_compressor_enabled=true   #enable qat
osd_class_dir="/usr/local/lib/rados-classes" # related to your ceph installation path
```

Info: `waikikibeach21=/dev/nvme0n1p1:/dev/nvme0n1p3:/dev/nvme1n1p1,/dev/nvme0n1p2:/dev/nvme0n1p4:/dev/nvme1n1p2` data type is determined by `disk_format=osd:data:db_wal`

Info: Please edit public_network, monitor_network, cluster_network according to your actual network!

#### Deploy

```bash
cd /opt/CeTune/deploy
run_deploy.py redeploy --gen_cephconf
```


#### Check

```bash
ceph -s
```

Successfull if health status is `HEALTH_WARN` or `HEALTH_OK`

Check device status:

```bash
$ lsblk
nvme0n1                   259:0    0   1.8T  0 disk
├─nvme0n1p1               259:1    0    50G  0 part
├─nvme0n1p2               259:2    0    50G  0 part
├─nvme0n1p3               259:3    0    50G  0 part
└─nvme0n1p4               259:4    0    50G  0 part
nvme1n1                   259:5    0   1.8T  0 disk
├─nvme1n1p1               259:6    0     5G  0 part /var/lib/ceph/mnt/osd-device-0-data
├─nvme1n1p2               259:7    0     5G  0 part /var/lib/ceph/mnt/osd-device-1-data
├─nvme1n1p3               259:8    0   500G  0 part
└─nvme1n1p4               259:9    0   500G  0 part
```

## Benchmark

### Bluestore Compression Test

#### Edit QAT Driver Config File

Need to modify QAT driver config files in /etc/c6xx_dev0.conf, /etc/c6xx_dev1.conf, /etc/c6xx_dev2.conf

Replace the User Process Instance Section with follows:

```conf
[SHIM]
NumberCyInstances = 0
NumberDcInstances = 8
NumProcesses = 4
LimitDevAccess = 0

# Data Compression - User instance #0
Dc0Name = "Dc0"
Dc0IsPolled = 1
# List of core affinities
Dc0CoreAffinity = 0

# Data Compression - User instance #1
Dc1Name = "Dc1"
Dc1IsPolled = 1
# List of core affinities
Dc1CoreAffinity = 1

# Data Compression - User instance #2
Dc2Name = "Dc2"
Dc2IsPolled = 1
# List of core affinities
Dc2CoreAffinity = 2

# Data Compression - User instance #3
Dc3Name = "Dc3"
Dc3IsPolled = 1
# List of core affinities
Dc3CoreAffinity = 3

# Data Compression - User instance #4
Dc4Name = "Dc4"
Dc4IsPolled = 1
# List of core affinities
Dc4CoreAffinity = 4

# Data Compression - User instance #5
Dc5Name = "Dc5"
Dc5IsPolled = 1
# List of core affinities
Dc5CoreAffinity = 5

# Data Compression - User instance #6
Dc6Name = "Dc6"
Dc6IsPolled = 1
# List of core affinities
Dc6CoreAffinity = 6

# Data Compression - User instance #7
Dc7Name = "Dc7"
Dc7IsPolled = 1
# List of core affinities
Dc7CoreAffinity = 7

```

#### Ceph Config

Add below config in Ceph config to enable qat:

```
bluestore_compression_mode=force
bluestore_compression_algorithm=zlib  # select zlib compression algorithm, this for all pool
qat_compressor_enabled=true   #enable qat
```
or set one pool :

```bash
ceph osd pool set rbd compression_algorithm zlib  #none,snappy,zstd,lz4
ceph osd pool set rbd compression_mode aggressive
ceph osd pool set rbd compression_required_ratio 0.875
```

#### Create Image
```bash
ceph osd pool create rbd 256
rbd pool init rbd
rbd create --size 10240 --image rbd/image --image-format 2 --thick-provision
```

#### Fio Config

```conf
[global]
ioengine=rbd
iodepth=128
rw=write
bs=4M
#size=100%
time_based=1
ramp_time=10m
runtime=20m
clientname=admin
pool=rbd
group_reporting
buffer_compress_percentage=80
refill_buffers
buffer_pattern=0xdeadbeef

[volumes]
rbdname=image
numjobs=1
```

#### Check QAT Device Activity

```bash
cat /sys/kernel/debug/qat_c6xx_0000:05:00.0/fw_counters # according to the path of your machine qat_c6xx_0000
```

These files store the QAT Acitivity information. 0 means there are no QAT activities during your benchmark

Here is an sample for the result
```bash
root@waikikibeach21:~# cat /sys/kernel/debug/qat_c6xx_0000:05:00.0/fw_counters
+------------------------------------------------+
| FW Statistics for Qat Device                   |
+------------------------------------------------+
| Firmware Requests [AE  0]:              260640 |
| Firmware Responses[AE  0]:              260640 |
| RAS Events        [AE  0]:                   0 |
+------------------------------------------------+
| Firmware Requests [AE  1]:              260635 |
| Firmware Responses[AE  1]:              260635 |
| RAS Events        [AE  1]:                   0 |
+------------------------------------------------+
| Firmware Requests [AE  2]:              260648 |
| Firmware Responses[AE  2]:              260648 |
| RAS Events        [AE  2]:                   0 |
+------------------------------------------------+
| Firmware Requests [AE  3]:              260640 |
| Firmware Responses[AE  3]:              260640 |
| RAS Events        [AE  3]:                   0 |
+------------------------------------------------+
| Firmware Requests [AE  4]:              260624 |
| Firmware Responses[AE  4]:              260624 |
| RAS Events        [AE  4]:                   0 |
+------------------------------------------------+
| Firmware Requests [AE  5]:              260649 |
| Firmware Responses[AE  5]:              260649 |
| RAS Events        [AE  5]:                   0 |
+------------------------------------------------+
| Firmware Requests [AE  6]:              260646 |
| Firmware Responses[AE  6]:              260646 |
| RAS Events        [AE  6]:                   0 |
+------------------------------------------------+
| Firmware Requests [AE  7]:              260629 |
| Firmware Responses[AE  7]:              260629 |
| RAS Events        [AE  7]:                   0 |
+------------------------------------------------+
| Firmware Requests [AE  8]:              260639 |
| Firmware Responses[AE  8]:              260639 |
| RAS Events        [AE  8]:                   0 |
+------------------------------------------------+
| Firmware Requests [AE  9]:              260646 |
| Firmware Responses[AE  9]:              260646 |
| RAS Events        [AE  9]:                   0 |
+------------------------------------------------+

```
