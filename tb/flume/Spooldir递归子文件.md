# Flume SpoolDir 使用

在 Flume 里面经常需要使用 SpoolDir 来监听某一个文件里面的文件变动.

但默认的实现是不能监听子文件夹的文件变动,就是说需要修改先关的源码才可以实现.

在现实生产环境中.flume 监听 ftp 文件夹,但 ftp 上传文件时,一直在追加,那么 flume 会报错.

所以 ftp 在上传文件的时候,添加`.tmp`后缀名,上传完重命名回来,那么样子就不会出错了.

---

## 1. flume 配置文件

```properties
hdfsAgent.sources = r1
hdfsAgent.channels = c1
hdfsAgent.sinks = s1

hdfsAgent.sources.r1.type = spooldir
hdfsAgent.sources.r1.batchSize=1000
hdfsAgent.sources.r1.pollDelay=3000
hdfsAgent.sources.r1.maxBackoff=5000
hdfsAgent.sources.r1.decodeErrorPolicy = IGNORE
hdfsAgent.sources.r1.fileSuffix=.COMPLETED
hdfsAgent.sources.r1.inputCharset=UTF-8
hdfsAgent.sources.r1.recursiveDirectorySearch=true
# 配置从ftp下载文件到本地的存放路径
hdfsAgent.sources.r1.spoolDir=/home/ftp/
hdfsAgent.sources.r1.ignorePattern = ^(.)*\\.tmp$
hdfsAgent.sources.r1.channels = c1
hdfsAgent.sources.r1.interceptors=i2
hdfsAgent.sources.r1.interceptors.i2.type=timestamp

#hdfsAgent.channels.c1.type=org.apache.flume.channel.DualChannel
#hdfsAgent.channels.c1.capacity=200000
#hdfsAgent.channels.c1.transactionCapacity=100000

hdfsAgent.channels.c1.type=org.apache.flume.channel.DualChannel
hdfsAgent.channels.c1.capacity=20000
hdfsAgent.channels.c1.transactionCapacity=10000

hdfsAgent.sinks.s1.type=hdfs
hdfsAgent.sinks.s1.channel=c1
# 配置hdfs存放数据的路径
hdfsAgent.sinks.s1.hdfs.path=hdfs://ns1/flume/data/%y-%m-%d
hdfsAgent.sinks.s1.hdfs.fileType=DataStream
hdfsAgent.sinks.s1.hdfs.writeFormat=Text
hdfsAgent.sinks.s1.hdfs.timeZone=Asia/Shanghai
hdfsAgent.sinks.s1.hdfs.rollInterval=0
hdfsAgent.sinks.s1.hdfs.rollSize=65011712
hdfsAgent.sinks.s1.hdfs.rollCount=0
hdfsAgent.sinks.s1.hdfs.idleTimeout=60
hdfsAgent.sinks.s1.hdfs.callTimeout =50000
```

---

## 2. 修改源码

修改:`org.apache.flume.client.avr.ReliableSpoolingFileEventReader`源码.

主要改动: `getNextFile()`和新增`getFilesByRecuris()`,同时应该注意 FileFilter 的相关条件改写.

详细如下代码所示:

```java
private Optional<FileInfo> getNextFile() {
		List<File> candidateFiles = Collections.emptyList();

		if (consumeOrder != ConsumeOrder.RANDOM || candidateFileIter == null || !candidateFileIter.hasNext()) {
			/* Filter to exclude finished or hidden files */

			/**
			 * 修改这里
			 *
			 * FileFilter filter = new FileFilter() { public boolean accept(File
			 * candidate) { String fileName = candidate.getName(); if
			 * ((candidate.isDirectory()) ||
			 * (fileName.endsWith(completedSuffix)) ||
			 * (fileName.startsWith(".")) ||
			 * ignorePattern.matcher(fileName).matches()) { return false; }
			 * return true; } };
			 *
			 */
			FileFilter filter = new FileFilter() {
				public boolean accept(File candidate) {
					String fileName = candidate.getName();
					if ((fileName.endsWith(completedSuffix)) || (fileName.startsWith("."))
							|| ignorePattern.matcher(fileName).matches()) {
						return false;
					}
					return true;
				}
			};

			// 这里面递归获取所有的文件

			List<File> allFiles = new ArrayList<File>();
			getFilesByRecurs(allFiles, filter, spoolDirectory);
			/**
			 * 主要修改这里
			 *
			 *
			 *
			 * candidateFiles = Arrays.asList(spoolDirectory.listFiles(filter));
			 */
			candidateFiles = allFiles;

			if (logger.isDebugEnabled()) {
				logger.info("All file {} size:{}", spoolDirectory.getAbsolutePath(), allFiles.size());
				// logger.info("All file: {}", allFiles);
			}
			listFilesCount++;
			candidateFileIter = candidateFiles.iterator();
		}

		if (!candidateFileIter.hasNext()) { // No matching file in spooling
											// directory.
			return Optional.absent();
		}

		File selectedFile = candidateFileIter.next();
		if (consumeOrder == ConsumeOrder.RANDOM) { // Selected file is random.
			return openFile(selectedFile);
		} else if (consumeOrder == ConsumeOrder.YOUNGEST) {
			for (File candidateFile : candidateFiles) {
				long compare = selectedFile.lastModified() - candidateFile.lastModified();
				if (compare == 0) { // ts is same pick smallest
									// lexicographically.
					selectedFile = smallerLexicographical(selectedFile, candidateFile);
				} else if (compare < 0) { // candidate is younger (cand-ts >
											// selec-ts)
					selectedFile = candidateFile;
				}
			}
		} else { // default order is OLDEST
			for (File candidateFile : candidateFiles) {
				long compare = selectedFile.lastModified() - candidateFile.lastModified();
				if (compare == 0) { // ts is same pick smallest
									// lexicographically.
					selectedFile = smallerLexicographical(selectedFile, candidateFile);
				} else if (compare > 0) { // candidate is older (cand-ts <
											// selec-ts).
					selectedFile = candidateFile;
				}
			}
		}

		return openFile(selectedFile);
	}


	/**
	 * 递归获取所有子文件夹文件
	 *
	 * @param files
	 *            文件列表
	 * @param filter
	 *            过滤器
	 * @param dir
	 *            文件路径
	 */
	private static void getFilesByRecurs(List<File> files, FileFilter filter, File dir) {
		try {
			for (File each : dir.listFiles(filter)) {
				if (each.isDirectory()) {
					getFilesByRecurs(files, filter, each);
				} else {
					if (each.isFile()) {
						files.add(each);
					}
				}
			}
		} catch (Exception e) {
			e.printStackTrace();
			logger.error("#getFilesByRecurs have an error:{}", e.getMessage());
		}
	}
```

---

## 3. 测试

把`class`更新进`flume-ng-core-1.6.0.jar`相关 package 里面,重启 flume 服务即可.
