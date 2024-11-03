# Perf toolbox

## Test environment

Here we explain the script ```./setup_test_env.sh```.

### The frequency

First, we have to set the frequency. On recent Linux distributions,
a pilot preventing the use of ```cpufreq-set``` is the default.
To disable it, append ```intel_pstate=disable``` to ```GRUB_CMDLINE_LINUX```
option in ```/etc/default/grub```, and reboot. It is shown
[here](https://stackoverflow.com/questions/23526671/how-to-solve-the-cpufreqset-errors).

Once it is done, you can set the frequency by doing:
```
sudo cpufreq-set -g userspace
sudo cpufreq-set -f 2.5Ghz
```
It is shown [here](https://www.thinkwiki.org/wiki/How_to_use_cpufrequtils).

### The huge pages

Then, we have to provide huge pages. We follow [this tutorial](https://paolozaino.wordpress.com/2016/10/02/how-to-force-any-linux-application-to-use-hugepages-without-modifying-the-source-code/).

First, install:
```
sudo apt install libhugelbfs-bin
apt install libhugetlbfs0
```

Check the Huge Page configuration:
```
$ grep HugePages_Total /proc/meminfo
HugePages_Total:   0
```

Then check the size of Huge Pages:
```
$ grep Hugepagesize /proc/meminfo
Hugepagesize:     2048 kB
```

Allocate 2048 Huge Pages (4GB of Huge Pages) in root:
```
# echo 2048 > /proc/sys/vm/nr_hugepages
```

Check the allocation with
```
$ grep HugePages_Total /proc/meminfo
$ grep HugePages_Free /proc/meminfo # how many are free
```

List all huge page pools available:
```
$ hugeadm --pool-list
```

Now we need to mount hugetlbfs (the chown is because the app is non-root;
of course, postfix is needed):
```
# mkdir -p /mnt/hugetlbfs
# mount -t hugetlbfs none /mnt/hugetlbfs
# chown postfix:postfix /mnt/hugetlbfs
```

Each mounting point is associated with a page size. To override the default:
```
# mkdir -p /mnt/hugetlbfs-64K
# mount -t hugetlbfs none -opagesize=64k /mnt/hugetlbfs-64K
# chown postfix:postfix /mnt/hugetlbfs-64k
```

Set the recommended Shared Memory Max:
```
# hugeadm --set-recommended-shmmax
```

Report of the configuration:
```
$ hugeadm --explain
```

Now you have just to launch your app (in our case, it is done in the
Python script when the flag ```use-huge-pages``` is set):
```
LD_PRELOAD=libhugetlbfs-2.23.so HUGETLB_MORECORE=yes <myapp>
```
The name ```libhugetlbfs-2.23.so``` can be found using
```apt-file list libhugetlbfs0```. For now it is hard-coded in 
```wrappers.py```. Feel free to modify.

## Binary analysis

Produce a pretty disassembled kernel:
```
objdump --disassemble=<myfunction> --disassembler-color=on --visualize-jumps=color --no-show-raw-insn --no-addresses -M intel64 <myapp>
```

Add the valuable ```perf``` annotations to the assembly:
```
perf record <myapp>
perf annotate -Mintel
```
