
This page describes how to use pbench-fio to do storage performance testing.  It includes:

* random and sequential workload parameters
* how to do cache dropping
* how to ramp up tests gradually for large host counts

Before we go into the details of the pbench-fio parameters, 
which are different in some cases than the fio input parameters, 
we describe why pbench-fio exists, and how to make use of it.

# overview of usage opportunities and constraints

pbench-fio was created to automate sets of fio tests, 
including calculations of statistics for throughputs and latency.  
The fio benchmark itself does not do this.

pbench-fio also was created with distributed storage performance testing in mind -- 
it uses the **fio --client** option to allow fio to start up a storage performance test 
on a set of hosts simultaneously and collect all the results to a central location.

pbench-fio supports ingestion of these results into a repository, 
such as elastic search, through JSON-formatted results that are easily parsed.  
Traditional fio output is very difficult to parse correctly 
and the resulting parser is hard to understand and likely to break as fio evolves over time.

pbench-fio also provides additional postprocessing for the new fio latency histogram logs.   This feature lets you graph latency percentiles cluster-wide as a function of time.  These graphs are really important when you are running very long tests and want to look at variations in latency caused by operational events such as node, disk or network failure/replacement.

# difference between sequential and random workload parameters

Ideally we would like to feed fio one set of test parameters 
and have it churn through all the multiple iterations of each data point, 
collecting and analyzing all the results for us.  That is the dream ;-)

However, here is why this is difficult or even impossible to do:

o applications often do sequential I/O in buffered mode
o applications often do random I/O in unbuffered mode

Unfortunately there is no one set of pbench-fio input parameters 
that lets you run both sequential and random tests with the same set of parameters, so far.

## sequential workloads

Applications typically do unbuffered I/O, 
with write aggregation and read prefetching done at client, using a single thread per file.  
Examples of this are software builds, and image/audio/video processing.  
If you do unbuffered (i.e. O_DIRECT) writes/reads to a sequential file, 
throughput will be extremely slow for small transfer sizes, 
but will not be representative of typical application performance.

## random workloads

High-performance applications sometimes do random I/O in O_DIRECT/O_SYNC mode 
(each write has to make it to storage before being acknowledged), 
using asynchronous I/O such as libaio library provides 
to maintain multiple in-flight I/O requests from a single thread.  
They do this for several reasons:

* avoid prefetching data that will never be used by the application
* avoid network overhead of small writes from the client to the server process

If you do buffered random writes, for example, 
you will queue up a ton of requests but they will not actually get written to disk immediately.  
This situation can result in artificially high IOPS numbers
which do not reflect steady-state performance of an application in this environment.  
There are a couple of ways to solve this problem:

1. run a test that is so large that the amount of buffered data is an insignificant percentage of the total data accessed.
2. run a write test that includes time required to fsync the file descriptors to flush all outstanding writes to block devices.

Both of these options have issues.  
Option 1 increases test time significantly 
depending on the potential amount of buffering taking place in the system. 
Option 2 has the disadvantages that short tests may spend excessive time 
in the fsync call and not in the actual write-related system calls, 
and queue depth is not controlled.

Contrast this with use of **O_DIRECT** or **O_SYNC** reads and writes.  
Here the situation is reversed - we can run a test that uses only a small 
fraction of the total data set and get representative numbers.  
For example, we could use **fallocate()** system call to quickly create files 
that span the entire block device, 
and then randomly read/write only an insignificant fraction of the entire set of files, 
yet still get representative throughput and latency numbers.  
The same is true for using fio on block devices.

# how to handle large host counts

If you are pushing limits of scalability with very large host counts, you have some additional tuning and workload parameters to consider.  It is very important that all fio processes and threads start to generate workload at the same time, and stop the workload at the same time.  Otherwise it is not valid to add the throughputs of these constituent threads to get the total system-wide workload, nor is it valid to obtain latency statistics!

## ARP cache increase

The default Linux parameters are sometimes too low for tests involving large numbers of VMs or containers, resulting in failure to cache IP addresses of all workload generates in the Linux ARP cache.  This can result in disconnected VMs.  To increase ARP cache size, see [this article](http://www.serveradminblog.com/2011/02/neighbour-table-overflow-sysctl-conf-tunning/) .  

## pdsh and ansible fanout

pdsh by default will only issue command in parallel to 32 hosts at a time.  Ansible will only issue command in parallel to 5 hosts by default.  This is good enough for many situations, but if you have a long running command that needs to run on more hosts in parallel than this, both pdsh and ansible have a **-f** parameter, but be careful with this because of the next problem.

## limit on simultaneous ssh sessions

ssh by default will not handle more than about 10 simultaneous ssh sessions started per second, but it can handle a very large number of concurrent sessions.  One tunable for sshd is the maximum number of sessions coming into a particular host, see [this article](http://unix.stackexchange.com/questions/22965/limits-of-ssh-multiplexing).

## fio startdelay parameter

It takes a while for fio to initiate workload on the entire set of hosts (much faster than ssh though).  We don't want the system getting busy while threads are being started!  Using the fio **startdelay=K** parameter (where K is number of seconds to wait before starting fio job) in your job file can allow the test driver to start fio workload generator processes without having them make the system busy right away, maximizing the chance of having all threads/processes start at approximately the same time.  

** fio ramp_time parameter

When cache has been dropped in a large cluster, the metadata kept in hosts' Linux buffer caches has been removed so it is not in its normal operating state anymore.  To let these metadata caches warm up a bit before unleashing the full workload, you can use the fio **ramp_time=K** parameter (where K is number of seconds to grow workload to full rate).  This is often used with random I/O tests, not sure of its applicability to sequential I/O tests.

FIXME: I'm guessing this has to be used with iops_rate=N parameter since otherwise fio would not know how to grow the IOPS rate.

# cache dropping 

In order to get reproducible, accurate results in a short amount of test time, you do not want to have client or server caching read data.  Why?  When doing performance tests, we want tests to cover a wide range of data points in a short time duration, but in real life, the cluster is run for months or even years without interruption, on amounts of data that are far greater than what we can access during the short performance test.  In order for our performance tests to match the steady-state throughput obtainable by the real users, we need to eliminate caching effects so that data is traversing the entire data path between block devices and application (Note: there are cases where it's ok to test the cache performance, as long as that is your intent!).  

To eliminate caching effects with pbench-fio, a new **--pre-iteration-script** option has been added.  This command is run by the test driver before each data point.  For example,  you could use the syntax **--pre-iteration-script=drop-cache.sh** , where this script looks like this:

    pdsh -S -w ^vms.list 'sync ; echo 3 > /proc/sys/vm/drop_caches'
    pdsh -S -w ^ceph-osd-nodes.list 'sync ; echo 3 > /proc/sys/vm/drop_caches'

This command should empty out the Linux buffer cache both on the guests and on the Ceph OSD hosts so that no data or metadata is cached there.  

The RBD cache is not cleared by this procedure, but the RBD cache is relatively small, 32 MB/guest, and so the significance of any caching there is not that great, particularly for workloads where you are using O_DIRECT and thereby bypassing the RBD cache.  To really flush the RBD cache, you would be forced to detach each inder volume and then re-attach it to its guest.  

For FUSE filesystems such as Gluster or Cephfs, the above procedure would not be sufficient in some cases and you might have to unmount and remount the filesystem, which is not a very time-consuming procedure.

Containers are more like processes than VMs, so that dropping Linux buffer cache on the node running the containers should be sufficient in most cases.

# warming up the system

If you drop cache, you need to allow the system to warm up its caches and reach steady-state behavior.  It is desirable for there to be no free memory either for most of test so that memory recycling behavior is tested.  So you may need considerable runtime at large scale.  You can specify maximum fio runtime using **runtime=k** line in job file where k is number of seconds.  Using pbench-fio you can observe the warmup behavior and see if your time duration needs adjustment.

For cases where the file can be completely read/written before the test duration has finished, if you don't want the test to stop, you must specify **time_based=1** to force fio to keep doing I/O to the file.  For example, on sequential files, if it reaches the end of file, it will seek to beginning of the file and start over.   On random I/O, it will keep generating random offsets even if the entire file has been accessed.

# rate-limiting IOPS

A successful test should show both high throughput and acceptable latency.  For random I/O tests, it is possible to bring the system to its knees by putting way more random I/O requests in flight than there are block devices to service them (i.e. increasing queue depth).  This can artificially drive up response time.  You can reduce queue depth by reducing **iodepth=k** parameter in the job file or reducing number of concurrent fio processes.  But another method that may be useful is to reduce the rate at which each fio process issues random I/O requests to a lower value than it would otherwise have.  You can do this with the **rate_iops=k** parameter in the job file.  For example, this was used to run a large cluster at roughly 3/4 of its throughput capacity for latency measurements.

# measuring latency percentiles as a function of time

Usually when a latency requirement is defined in a SLA (service-level agreement) for a cluster, the implication is that *at any point in time* during the operation of the cluster, the Nth latency percentile will not exceed X seconds.  However, fio does not output that information directly in its normal logs - instead it outputs latency percentiles measured over the duration of the entire test run.  For short test this may be sufficient, but for longer tests or tests designed to measure *changes* in latency, you need to capture the variation in latency percentiles over time.   The new fio histogram logging feature helps you do that.  It uses the histogram data structures internal to every fio process, emitting this histogram to a log file at a user-specified interval.  A post-processing tool, [fiologparser-hist.py](https://github.com/axboe/fio/blob/master/tools/hist/fiologparser_hist.py), then reads in these histogram logs and outputs latency percentiles over time.   pbench's postprocessing tool runs it and then graphs the results.  The job file parameters **write_hist_log=h** and **hist_log_msec** control the pathnames and interval for histogram logging.

# example of use case for OpenStack Cinder volumes

Advice: do everything on smallest possible scale to start with, until you get your input files to work, then scale it out.

construct a list of virtual machine IP addresses which are reachable from your test driver with ssh using no password.  These virtual machines should all have a cinder volume attached (i.e. /dev/vdb inside VM).  You should format the cinder volume from inside the VM with a Linux filesystem such as XFS

    testdriver# export PDSH_RCMD_TYPE=ssh
    testdriver# alias mypdsh="pdsh -S -w ^vms.list"

make sure it's not already mounted

    testdriver# mypdsh 'umount /mnt/fio'

For OpenStack, it's a good idea to preallocate RBD images so that you get consistent performance results.  To do this, you can dd to /dev/vdb like this (it could take a while):

    testdriver# mypdsh -f 128 'dd if=/dev/zero of=/dev/vdb bs=4096k conv=fsync'
    
Now format the filesystems for each cinder volume and mount them:

    testdriver# mypdsh -f 128 'mkfs -t xfs /dev/vdb && mkdir -p /mnt/fio && mount -t xfs -o noatime /dev/vdb /mnt/fio && mkdir /mnt/fio/files'
           
Then you construct a fio job file for your initial sequential tests, and this will also create the files for the subsequent tests.  certain parameters have to be specified with the string `$@` because pbench-fio wants to fill them in.   It might look something like this:

`[global]
# size of each FIO file (in MB)
size=$@
# do not use gettimeofday, much too expensive for KVM guests
clocksource=clock_gettime
# give fio workers some time to get launched on remote hosts
startdelay=5
# files accessed by fio are in this directory
directory=$@
# write fio latency logs in /var/tmp with "fio" prefix
write_lat_log=/var/tmp/fio
# write a record to latency log once per second
log_avg_msec=1000
# write fio histogram logs in /var/tmp/ with "fio" prefix
write_hist_log=/var/tmp/fio
# write histogram record once every 10 seconds
log_hist_msec=10
# only one process per host
numjobs=1
# do an fsync on file before closing it (has no effect for reads)
fsync_on_close=1
# what's this?
per_job_logs=1

[sequential]
rw=$@
bs=$@

# do not lay out the file before the write test, 
# create it as part of the write test
create_on_open=1
# do not create just one file at a time
create_serialize=0`

And you write it to a file named fio-sequential.job, then run it with a command like this one, which launches fio on 1K guests with json output format.

/usr/local/bin/fio --client-file=vms.list --pre-iteration-script=drop-cache.sh --rw=write,read -b 4,128,1024 -d /mnt/fio/files --max-jobs=1024 --output-format=json fio-sequential.job

This will write the files in parallel to the mount point.  The sequential read test that follows can use the same job file.  The **--max-jobs** parameter should match the count of the number of records in the vms.list file (FIXME: is --max-jobs still needed?).

Since we are using buffered I/O, we can usually get away with using a small transfer size, since the kernel will do prefetching, but there are exceptions, and you may need to vary the **bs** parameter.

For random writes, the job file is a little more complicated.  Again the `[global]` section is the same but the workload-specific part needs to be more complex to express some new parameters:

`[random]
rw=$@
bs=$@
# specify use of libaio 
# to allow a single thread to launch multiple parallel I/O requests
ioengine=libaio
# how many parallel I/O requests should be attempted
iodepth=4
# use O_DIRECT flag when opening a file
direct=1
# when the test is done and the file is closed, do an fsync first
fsync_on_close=1
# if we finish randomly accessing entire file, keep going until time is up
time_based=1
# test for 1/2 hour (gives caches time to warm up on large cluster)
runtime=1800`

The random read test can be performed using the same job file if you want.

/usr/local/bin/fio --client-file=vms.list --pre-iteration-script=drop-cache.sh --rw=write,read -b 4,128,1024 -d /mnt/fio/files --max-jobs=1024 --output-format=json fio-random.job

