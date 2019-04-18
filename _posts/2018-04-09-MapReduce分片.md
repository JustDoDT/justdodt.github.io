---
layout:     post
title:      "MapReduce分片"
date:       2018-04-09 23:01:00
author:     "JustDoDT"
header-img: "img/haha.jpg"
catalog: true
tags:
    - hadoop, mapreduce
---

### 概述
在进行map计算之前，map会根据输入文件计算输入分片（input split）,每个输入分片（input split）针对一个map任务，输入分片（input split）存储的并非数据本身，而是一个分片长度和一个记录数据的位置的数组。

### 问题
Hadoop 2.x默认的block大小是128MB，Hadoop 1.x默认的block大小是64MB，可以在hdfs-site.xml中设置dfs.block.size，注意单位是byte。

分片大小范围可以在mapred-site.xml中设置，mapred.min.split.size mapred.max.split.size，minSplitSize大小默认为1B，maxSplitSize大小默认为Long.MAX_VALUE = 9223372036854775807

**那么分片到底是多大呢？**

minSize=max{minSplitSize,mapred.min.split.size} 

maxSize=mapred.max.split.size

splitSize=max{minSize,min{maxSize,blockSize}}

看一下源码：
[源码地址](https://github.com/apache/hadoop/blob/2addebb94f592e46b261e2edd9e95a82e83bd761/hadoop-mapreduce-project/hadoop-mapreduce-client/hadoop-mapreduce-client-core/src/main/java/org/apache/hadoop/mapred/FileInputFormat.java)

    long blockSize = file.getBlockSize();
    long splitSize = computeSplitSize(goalSize, minSize, blockSize);
          
       .
       .
       .
    protected long computeSplitSize(long goalSize, long minSize,
                                       long blockSize) {
    return Math.max(minSize, Math.min(goalSize, blockSize));
          
   
   
所以在我们没有设置分片的范围的时候，分片大小是由block块大小决定的，和它的大小一样。比如把一个258MB的文件上传到HDFS上，假设block块大小是128MB，那么它就会被分成三个block块，与之对应产生三个split，所以最终会产生三个map task。我又发现了另一个问题，第三个block块里存的文件大小只有2MB，而它的block块大小是128MB，
**那它实际占用Linux file system的多大空间？**

答案是实际的文件大小，而非一个块的大小

**最后一个问题是**： 如果hdfs占用Linux file system的磁盘空间按实际文件大小算，那么这个”块大小“有必要存在吗？

其实块大小还是必要的，一个显而易见的作用就是当文件通过append操作不断增长的过程中，可以通过来block size决定何时split文件。

    
   
  [源码地址](https://github.com/apache/hadoop/blob/2addebb94f592e46b261e2edd9e95a82e83bd761/hadoop-mapreduce-project/hadoop-mapreduce-client/hadoop-mapreduce-client-core/src/main/java/org/apache/hadoop/mapred/FileInputFormat.java) 

**源码解读**

        computeSplitSize
              /** 计算切片大小
              *		blockSize 
              *		minSize 1
              *		maxSize 9223372036854775807
              */
              protected long computeSplitSize(long blockSize, long minSize,
                                              long maxSize) {
                                              //  maxSize blockSize 取最小是 blockSize 
                                              //然后 minSize 与 blockSize 取最大 结果是  blockSize
                return Math.max(minSize, Math.min(maxSize, blockSize));
              }
        
        
        
          /** 
           * 获取splits集合
           */
          public List<InputSplit> getSplits(JobContext job) throws IOException {
            StopWatch sw = new StopWatch().start();
            long minSize = Math.max(getFormatMinSplitSize(), getMinSplitSize(job)); //1
            long maxSize = getMaxSplitSize(job);//9223372036854775807
        
            // 创建splits集合
            List<InputSplit> splits = new ArrayList<InputSplit>();
            //得到hdfs文件列表
            List<FileStatus> files = listStatus(job);
            //对文件列表进行遍历
            for (FileStatus file: files) {
              Path path = file.getPath();//路径
              long length = file.getLen();//文件长度
              if (length != 0) {
              //文件块的文位置
                BlockLocation[] blkLocations;
                if (file instanceof LocatedFileStatus) {
                  blkLocations = ((LocatedFileStatus) file).getBlockLocations(); //得到文件块的位置
                } else {
                  FileSystem fs = path.getFileSystem(job.getConfiguration());
                  blkLocations = fs.getFileBlockLocations(file, 0, length);
                }
                //判断文件是否可切分
                if (isSplitable(job, path)) {
                  long blockSize = file.getBlockSize();//33554432
                  long splitSize = computeSplitSize(blockSize, minSize, maxSize); //blockSize
        
                  long bytesRemaining = length;
                   private static final double SPLIT_SLOP = 1.1;   // 10% slop
                  当文件长度/splitsize >    1.1 时 
                  while (((double) bytesRemaining)/splitSize > SPLIT_SLOP) {
                    int blkIndex = getBlockIndex(blkLocations, length-bytesRemaining);
                    splits.add(makeSplit(path, length-bytesRemaining, splitSize,
                                blkLocations[blkIndex].getHosts(),
                                blkLocations[blkIndex].getCachedHosts()));//splits添加分片
                    bytesRemaining -= splitSize;
                  }
                  当还有剩余的文件
                  if (bytesRemaining != 0) {
                int blkIndex = getBlockIndex(blkLocations, length-bytesRemaining);
                splits.add(makeSplit(path, length-bytesRemaining, bytesRemaining,
                           blkLocations[blkIndex].getHosts(),
                           blkLocations[blkIndex].getCachedHosts()));//加入集合
              }
            } else { // 不可切分
              splits.add(makeSplit(path, 0, length, blkLocations[0].getHosts(),
                          blkLocations[0].getCachedHosts()));
            }
          } else { 
            //Create empty hosts array for zero length files
            //文件长度为0时创建空文件
            splits.add(makeSplit(path, 0, length, new String[0]));
          }
        }
        
                ......
                //返回切片集合
            return splits;
          }
        	protected boolean isSplitable(JobContext context, Path filename) {
        		return true;
        	}
        	protected FileSplit makeSplit(Path file, long start, long length, 
                                    String[] hosts, String[] inMemoryHosts) {
        	return new FileSplit(file, start, length, hosts, inMemoryHosts);
        }



**通过上面的源码可以得出，加入一个块大小设置为128MB，如果输入的文件小于128*1.1=140.8MB; 那么就只有一个分片;如果输入的文件大于 140.8MB，小于128*2.1MB，就有2个分片。**

### 测试
测试数据 data1.log  139MB < 128*1.1 =140.8MB


        [hadoop@hadoop001 data]$ du -sh data1.log 
        139M	data1.log
        
        [hadoop@hadoop001 lib]$ hadoop jar g6-hadoop-1.0_back.jar com.ruozedata.hadoop.mapreduce.driver.LogETLDriver  /hadoop/split             /hadoop/split/output1
        
        [hadoop@hadoop001 lib]$ hadoop jar g6-hadoop-1.0_back.jar com.ruozedata.hadoop.mapreduce.driver.LogETLDriver  /hadoop/split             /hadoop/split/output1
        19/03/30 16:05:00 INFO client.RMProxy: Connecting to ResourceManager at /0.0.0.0:8032
        19/03/30 16:05:01 WARN mapreduce.JobResourceUploader: Hadoop command-line option parsing not performed. Implement the Tool               interface and execute your application with ToolRunner to remedy this.
        19/03/30 16:05:01 INFO input.FileInputFormat: Total input paths to process : 1
        19/03/30 16:05:01 INFO lzo.GPLNativeCodeLoader: Loaded native gpl library from the embedded binaries
        19/03/30 16:05:01 INFO lzo.LzoCodec: Successfully loaded & initialized native-lzo library [hadoop-lzo rev                               f1deea9a313f4017dd5323cb8bbb3732c1aaccc5]
        19/03/30 16:05:01 INFO mapreduce.JobSubmitter: number of splits:1

**注意：number of splits:1**
![一个分片测试](/img/MR1.png)

测试数据 data2.log 148M	 > 128 * 1.1 MB 

        [hadoop@hadoop001 data]$ du -sh data2.log 
        148M	data2.log
        [hadoop@hadoop001 data]$ hdfs dfs -put data2.log /hadoop/split


        [hadoop@hadoop001 lib]$ hadoop jar g6-hadoop-1.0_back.jar com.ruozedata.hadoop.mapreduce.driver.LogETLDriver                         /hadoop/split/data2.log  /hadoop/split/output2
        19/03/30 16:10:32 INFO client.RMProxy: Connecting to ResourceManager at /0.0.0.0:8032
        19/03/30 16:10:33 WARN mapreduce.JobResourceUploader: Hadoop command-line option parsing not performed. Implement the Tool  interface and execute your application with ToolRunner to remedy this.
        19/03/30 16:10:33 INFO input.FileInputFormat: Total input paths to process : 1
        19/03/30 16:10:33 INFO lzo.GPLNativeCodeLoader: Loaded native gpl library from the embedded binaries
        19/03/30 16:10:33 INFO lzo.LzoCodec: Successfully loaded & initialized native-lzo library [hadoop-lzo rev f1deea9a313f4017dd5323cb8bbb3732c1aaccc5]
        19/03/30 16:10:33 INFO mapreduce.JobSubmitter: number of splits:2



![两个分片测试](/img/MR2.png)


### 总结：
Split 分片的计算跟 blockSize, minSize, maxSize 三个参数有关
假如 blockSize 设置128 M
文件大小 200M
那么splitSize 就是128M
200/128=1.56>1.1 创建2个split （通过源码可以看出最右一个文件的大小在 0- 128+12.8M之间）

          
