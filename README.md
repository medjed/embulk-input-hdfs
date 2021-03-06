# Hdfs file input plugin for Embulk
[![Build Status](https://travis-ci.org/civitaspo/embulk-input-hdfs.svg)](https://travis-ci.org/civitaspo/embulk-input-hdfs)
[![Coverage Status](https://coveralls.io/repos/civitaspo/embulk-input-hdfs/badge.svg?branch=master&service=github)](https://coveralls.io/github/civitaspo/embulk-input-hdfs?branch=master)

Read files on Hdfs.

## Overview

* **Plugin type**: file input
* **Resume supported**: not yet
* **Cleanup supported**: no

## Configuration

- **config_files** list of paths to Hadoop's configuration files (array of strings, default: `[]`)
- **config** overwrites configuration parameters (hash, default: `{}`)
- **path** file path on Hdfs. you can use glob and Date format like `%Y%m%d/%s` (string, required).
- **rewind_seconds** When you use Date format in input_path property, the format is executed by using the time which is Now minus this property. (long, default: `0`)
- **partition** when this is true, partition input files and increase task count. (boolean, default: `true`)
- **num_partitions** number of partitions. (long, default: `Runtime.getRuntime().availableProcessors()`)
- **skip_header_lines** Skip this number of lines first. Set 1 if the file has header line. (long, default: `0`)
- **decompression** Decompress compressed files by hadoop compression codec api. (boolean. default: `false`)

## Example

```yaml
in:
  type: hdfs
  config_files:
    - /etc/hadoop/conf/core-site.xml
    - /etc/hadoop/conf/hdfs-site.xml
  config:
    fs.defaultFS: 'hdfs://hadoop-nn1:8020'
    dfs.replication: 1
    fs.hdfs.impl: 'org.apache.hadoop.hdfs.DistributedFileSystem'
    fs.file.impl: 'org.apache.hadoop.fs.LocalFileSystem'
  path: /user/embulk/test/%Y-%m-%d/*
  rewind_seconds: 86400
  partition: true
  num_partitions: 30
  decoders:
    - {type: gzip}
  parser:
    charset: UTF-8
    newline: CRLF
    type: csv
    delimiter: "\t"
    quote: ''
    escape: ''
    trim_if_not_quoted: true
    skip_header_lines: 0
    allow_extra_columns: true
    allow_optional_columns: true
    columns:
    - {name: c0, type: string}
    - {name: c1, type: string}
    - {name: c2, type: string}
    - {name: c3, type: long}
```

## Note
- The parameter **num_partitions** is the approximate value. The actual num_partitions is larger than this parameter.
  - see: [The Partitioning Logic](#partition_logic)
- the feature of the partition supports only 3 line terminators.
  - `\n`
  - `\r`
  - `\r\n`

## The Reference Implementation
- [hito4t/embulk-input-filesplit](https://github.com/hito4t/embulk-input-filesplit)

##<a id="partition_logic">The Partitioning Logic</a>

```
int partitionSizeByOneTask = totalFileLength / approximateNumPartitions;

/*
...
*/
            long numPartitions = 1; // default is no partition.
            if (isPartitionable(task, conf, status)) { // partition: true and (decompression: false or CompressionCodec is null)
                numPartitions = ((status.getLen() - 1) / partitionSizeByOneTask) + 1;
            }

            for (long i = 0; i < numPartitions; i++) {
                long start = status.getLen() * i / numPartitions;
                long end = status.getLen() * (i + 1) / numPartitions;
                if (start < end) {
                    TargetFileInfo targetFileInfo = new TargetFileInfo.Builder()
                            .pathString(status.getPath().toString())
                            .start(start)
                            .end(end)
                            .isDecompressible(isDecompressible(task, conf, status))
                            .isPartitionable(isPartitionable(task, conf, status))
                            .numHeaderLines(task.getSkipHeaderLines())
                            .build();
                    builder.add(targetFileInfo);
                }
            }
/*
...
*/

```


## Build

```
$ ./gradlew gem
```

## Development

```
$ ./gradlew classpath
$ bundle exec embulk run -I lib example.yml
```
