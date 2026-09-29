# ScaleCache: A Scalable Page Cache for Multiple Solid-State Drives

This repository contains the Linux 6.6.8 version of ScaleCache, its pxt4 file system module, and the workloads used for evaluation.

## Repository structure

```text
ScaleCache-6.6.8/
├── linux-6.6.8/        # ScaleCache Linux kernel
├── pxt4/               # ext4 with ScaleCache, including jbd3
└── workloads/
    ├── ffsb/
    ├── filebench/
    ├── fio/
    └── YCSB_rocksdb/
```

Run the commands below from the repository root unless a different directory is specified.

## 1. Build and install the kernel

```bash
cd linux-6.6.8
cp config-scalecache .config
make oldconfig

make -j "$(nproc)"
make -j "$(nproc)" modules
sudo make -j "$(nproc)" modules_install INSTALL_MOD_STRIP=1
sudo make -j "$(nproc)" install
cd ..
```

Reboot into the installed kernel before building pxt4. Check the running version with `uname -r`.

## 2. Build and load pxt4

Build and load `jbd3` first, then build and load `pxt4`:

```bash
cd pxt4/jbd3
make -j "$(nproc)"
sudo insmod jbd3.ko
cp Module.symvers ../

cd ..
make -j "$(nproc)"
sudo insmod pxt4.ko

cd ..
```

## 3. Prepare the RAID device and mount pxt4

The evaluation server uses **eight FireCuda NVMe drives** in a RAID 0 array. Identify all eight drives before running these commands. **Do not include an Intel NVMe drive.** Stopping an existing array, creating a new array, and formatting it can destroy data.

The drive identification script is located in the separate ScaleSwap repository:

```bash
cd /path/to/ScaleSwap/scripts
./check_model
```

Stop the existing arrays if they are active, then replace the eight example device paths with the FireCuda device paths reported on your server:

```bash
sudo mdadm --stop /dev/md127
sudo mdadm --stop /dev/md126

sudo mdadm --create /dev/md127 --raid-devices=8 --level=0 \
  /dev/nvmeXn1 /dev/nvmeYn1 /dev/nvmeZn1 /dev/nvmeAn1 \
  /dev/nvmeBn1 /dev/nvmeCn1 /dev/nvmeDn1 /dev/nvmeEn1

sudo mkfs -t ext4 \
  -E lazy_itable_init=0,lazy_journal_init=0 \
  -O ^has_journal \
  /dev/md127
```

Return to the ScaleCache repository and mount the array at `/mnt/test`, which is the path used by the supplied Filebench and YCSB scripts:

```bash
cd /path/to/ScaleCache-6.6.8
sudo mkdir -p /mnt/test
sudo mount -t pxt4 /dev/md127 /mnt/test
findmnt --mountpoint /mnt/test
```

## 4. Run the workloads

### fio: microbenchmark

```bash
cd workloads/fio
sudo fio rnd_wr.fio
```

### FFSB: macrobenchmark

The `ffsb` executable takes a configuration file as its argument. The supplied profile is named `test`. Check its file-system location before running it and set that location to the mounted test directory if needed.

```bash
cd workloads/ffsb
./configure
make -j "$(nproc)"
sudo ./ffsb test
```

### Filebench: fileserver and varmail macrobenchmarks

On systems with Automake 1.16, the checkout may contain helper symlinks that still point to Automake 1.15. If `./configure` reports that it cannot find `install-sh` or cannot run `config.sub`, repair the broken symlinks:

```bash
cd workloads/filebench

for f in install-sh missing config.sub config.guess depcomp ylwrap; do
  if [ -L "$f" ] && [ ! -e "$f" ] &&
     [ -f "/usr/share/automake-1.16/$f" ]; then
    ln -sfn "/usr/share/automake-1.16/$f" "$f"
  fi
done

./configure
make -j "$(nproc)"
```

The supplied `fileserver.f` profile uses `/mnt/test`. Check the directory setting in `varmail.f` as well. The fileserver profile defines two preallocated filesets of 64 files × 3 GiB each, so allow approximately **384 GiB** for its initial data.

Run Filebench with ASLR disabled for the process. This avoids the segmentation fault observed with this build:

```bash
sudo setarch "$(uname -m)" -R ./filebench -f fileserver.f
sudo setarch "$(uname -m)" -R ./filebench -f varmail.f
```

Run the two profiles separately. The `fileserver.f` profile contains `run 60`.

### YCSB with RocksDB: macrobenchmark

This repository already contains YCSB under `workloads/YCSB_rocksdb`; a separate YCSB release download is unnecessary. Build the RocksDB binding and its dependencies:

```bash
cd workloads/YCSB_rocksdb
mvn -Psource-run -pl site.ycsb:rocksdb-binding -am package -DskipTests
```

The supplied runner scripts must call `bin/ycsb.sh`. Their original `bin/ycsb` launcher uses Python 2 syntax and fails when run with Python 3. Update the active commands in both scripts:

```bash
sed -i '/^[[:space:]]*sudo \.\/bin\/ycsb /s#\./bin/ycsb #./bin/ycsb.sh #' \
  run_ycsb_workloada.sh run_ycsb_workloadf.sh
```

Both scripts set `rocksdb.dir=/mnt/test`. Confirm that pxt4 is mounted there, then run the desired workload:

```bash
findmnt --mountpoint /mnt/test

# Workload A: one load followed by five runs
bash -e ./run_ycsb_workloada.sh

# Workload F: one load followed by five runs
bash -e ./run_ycsb_workloadf.sh
```

The supplied `workloads/workloadf` currently sets `recordcount=1000`, `operationcount=1000`, and `threadcount=64`. Review these values before using its results for a larger evaluation. Run Workloads A and F with separate RocksDB data directories, or clear the test data between experiments, if each workload must start from a fresh database.
