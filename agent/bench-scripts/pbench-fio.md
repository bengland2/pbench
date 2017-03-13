This page describes how to use pbench-fio to do storage performance testing.  
Before we go into the details of the pbench-fio parameters, 
which are different in some cases than the fio input parameters, 
we describe why pbench-fio exists, and how to make use of it.

pbench-fio was created to automate sets of fio tests, 
including calculations of statistics for throughputs and latency.  
The fio benchmark itself does not do this.

pbench-fio also was created with distributed storage performance testing in mind -- 
it uses the "fio --client" option to allow fio to start up a storage performance test 
on a set of hosts simultaneously and collect all the results to a central location.

pbench-fio supports ingestion of these results into a repository, 
such as elastic search, through JSON-formatted results that are easily parsed.  
Traditional fio output is very difficult to parse correctly 
and the resulting parser is hard to understand and likely to break as fio evolves over time.

So ideally we would like to feed fio one set of test parameters 
and have it churn through all the multiple iterations of each data point, 
collecting and analyzing all the results for us.  That is the dream ;-)

However, here is why this is difficult or even impossible to do:

o applications often do sequential I/O in buffered mode
o applications often do random I/O in unbuffered mode

Unfortunately there is no one set of pbench-fio input parameters 
that lets you run both sequential and random tests with the same set of parameters, so far.

= sequential workloads

Applications typically do unbuffered I/O, 
with write aggregation and read prefetching done at client, using a single thread per file.  
Examples of this are software builds, and image/audio/video processing.  
If you do unbuffered (i.e. O_DIRECT) writes/reads to a sequential file, 
throughput will be extremely slow for small transfer sizes, 
but will not be representative of typical application performance.

= random workloads

High-performance applications sometimes do random I/O in O_DIRECT/O_SYNC mode 
(each write has to make it to storage before being acknowledged), 
using asynchronous I/O such as libaio library provides 
to maintain multiple in-flight I/O requests from a single thread.  
They do this for several reasons:

- avoid prefetching data that will never be used by the application
- avoid network overhead of small writes from the client to the server process

If you do buffered random writes, for example, 
you will queue up a ton of requests but they will not actually get written to disk immediately.  
This situation can result in artificially high IOPS numbers
which do not reflect steady-state performance of an application in this environment.  
There are a couple of ways to solve this problem:

1) run a test that is so large that the amount of buffered data is an insignificant percentage of the total data accessed.
2) run a write test that includes time required to fsync the file descriptors to flush all outstanding writes to block devices.

Both of these options have issues.  
Option 1 increases test time significantly 
depending on the potential amount of buffering taking place in the system. 
Option 2 has the disadvantages that short tests may spend excessive time 
in the fsync call and not in the actual write-related system calls, 
and queue depth is not controlled.

Contrast this with use of O_DIRECT reads and writes.  
Here the situation is reversed - we can run a test that uses only a small 
fraction of the total data set and get representative numbers.  
For example, we could use fallocate() system call to quickly create files 
that span the entire block device, 
and then randomly read/write only an insignificant fraction of the entire set of files, 
yet still get representative throughput and latency numbers.  
The same is true for using fio on block devices.
