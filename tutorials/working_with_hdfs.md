# Working with HDFS

## Installing Hadoop

### Download Hadoop

Go to Hadoop [releases page](http://hadoop.apache.org/releases.html) and follow the instructions to download the latest version (v3.1.2).

### Install Required Software

#### Install and configure SSH

ssh must be installed and sshd must be running to use the Hadoop scripts that manage remote Hadoop daemons if the optional start and stop scripts are to be used.
Additionally, it is recommmended that pdsh also be installed for better ssh resource management.

To install ssh and pdsh, run the following:

```
  $ sudo apt-get install ssh
  $ sudo apt-get install pdsh
```

Hadoop needs to use ssh without requiring a passphrase. To check if a passphrase is required, run:

```
  $ ssh localhost
```

If you are prompted to provide a passphrase, then run the following:

```
  $ ssh-keygen -t rsa -P '' -f ~/.ssh/id_rsa
  $ cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
  $ chmod 0600 ~/.ssh/authorized_keys
```
Now you should be able to connect to ssh without requiring any user interaction.

If pdsh remote command (RCMD) type is not set to ssh, you need to set it.
You can check the remote command type using:

```
  $ echo $PDSH_RCMD_TYPE
```

If the result is `ssh`, then your system is properly configured, otherwise, run the following:

```
  $ export PDSH_RCMD_TYPE=ssh
```

You can also permanently add export `PDSH_RCMD_TYPE` environment variable. This can be done by editing
`/etc/environment` file on ubuntu for instance.

#### Install Java

If Java is not installed on your machine. Go on and install it.

This is a good [guide](https://www.digitalocean.com/community/tutorials/how-to-install-java-with-apt-on-ubuntu-18-04) for `Ubuntu 18.04`.

Once Java is installed, you need to set `JAVA_HOME` environment variable.

Check if `JAVA_HOME` is set by:

```
  $ echo $JAVA_HOME
```

### Configure HDFS

We will be setting a sandbox single node in a pseudo-distributed mode where each Hadoop daemon runs in a separate Java process.

**Host and port number**

We need first to set the HDFS file system hostname and port number. 

Get the hotname by running:

```
$ hostname
```

Open `./etc/hadoop/core-site.xml` and add the following:

```
<configuration>
    <property>
        <name>fs.defaultFS</name>
        <value>hdfs://<hostname>:9000</value>
    </property>
</configuration>
```

**Replication Factor**


In a single node, the HDFS replication factor must be set to 1. Open `./etc/hadoop/hdfs-site.xml` and add the following:

```
<configuration>
    <property>
        <name>dfs.replication</name>
        <value>1</value>
    </property>
</configuration>
```

**Hadoop Environment**

You need to set some Hadoop environment variables before starting to use HDFS. Open `./etc/hadoop/hadoop-env.sh`,
locate the line exporting `JAVA_HOME`, uncomment the line and update it to match your java home.

This is also the placeholder for Hadoop home and logs folders. You can update them to customise the installation.
In our case, we will use the default values. In particular, all Hadoop logs will be stored in `HADOOP_HOME/logs` folder. To override this value, uncomment and update the following line:

```
# export HADOOP_LOG_DIR=${HADOOP_HOME}/logs
```

### Starting the Namenode

**Format the filesystem**

In order to format the HDFS file system, the `namenode` command needs to be used as follows:

```
$ ./bin/hdfs namenode -format
``` 

By default the HDFS filesystem will be set into your `/tmp` folder. This can be changed in the configuration file.
This can be done by editing `./etc/hadoop/hdfs-site.xml` file and adding the following properties in the configuration:

```
    <property>
        <name>dfs.name.dir</name>
        <value>file://<name node metadata folder></value>
    </property>

    <property>
        <name>dfs.data.dir</name>
        <value>file://<data node data folder></value>
    </property>

```

Please replace `<name node metadata folder>` and `<data node data folder>` with your folders of choice.

**Start the Namenode**

Once the HDFS file system is formatted, you can start the `NameNode` and `DataNode` deamon as follows:

```
$ ./sbin/start-dfs.sh
```

The `NameNode` daemon log output will be written to the `$HADOOP_LOG_DIR` directory.
If the `HADOOP_LOG_DIR` environment variable is not configured, it will default to `HADOOP_HOME/logs`.

Once the NameNode is up and running, its Web interface will be available at: `http://localhost:9870/`.

Now let's create the HDFS directories for the user:

```
$ ./bin/hdfs dfs -mkdir /user
$ ./bin/hdfs dfs -mkdir /user/<username>
```

MapReduce jobs will use the user's HDFS directories.

Wait few minutes and check the logs of the Data Node and Name Node under `HADOOP_HOME/logs/`:

```
  $ more ./logs/hadoop-<username>-namenode-<hostname>.log
  $ more ./logs/hadoop-<username>-datanode-<hostname>.log
```

## Using HDFS

### Using hdfs dfs command line

**Add Hadoop bin to your PATH**

It is more convenient to add Hadoop's bin to your PATH environment variable.

```
export PATH=PATH_TO_HADOOP_INSTALLATION_FOLDER/bin:$PATH
```

You can also add it parmanently to your profile.

Now you can call `hdfs` directly.

**Create an HDFS folder**

To create a folder use:

```
$ hdfs dfs -mkdir myfolder
```

This will create a fodler `myfolder` in your user's HDFS directory.

**Get directory listing**

You can list files in your user's home directory using:

```
$ hdfs dfs -ls
```

To list files in the HDFS root directory:

```
$ hdfs dfs -ls /
```

Or, to list files of a particular user:

```
$ hdfs dfs -ls /user/<username>/
```

This will give the same output as with listing files in the user's home directory.

It is also practical to list the contents of a directory `recursively` using:

```
$ hdfs dfs -ls -R /user/<username>/
```

**Copy files to HDFS**

```
$ hdfs dfs -put test.txt myfolder
```

This will copy file `test.txt` from the local filesystem to HDFS `myfolder` directory.
The HDFS full path of the copied file will be: */user/<username>/myfolder/test.txt*.

You can also use `wildcards` to copy all files matching some criteria as follows:

```
$ hdfs dfs -put *.txt myfolder
```

This will copy all files with .txt extension into myfolder HDFS directory.

**Display file content**

You can display the contents of an HDFS file using:

```
hdfs dfs -cat /user/<username>/myfolder/test.txt
```

Often you just want to check the `head` or `tail` (first or last lines) of a file. This can be done as follows:

```
hdfs dfs -head /user/<username>/myfolder/test.txt

hdfs dfs -tail /user/<username>/myfolder/test.txt
```

**Display statistics about file/directory**

You can display statistics about DHFS files and directories using:

```
hdfs dfs -stat %b /user/<username>/myfolder/test.txt
```

This prints the size in Bytes of the file. The `format` option *%b* is for the size in bytes. The following format options are available:
* %a and %A for permission
* %b for filesize
* %F for file type (file or directory)
* %g for group name of owner
* %n for filename
* %o for block size
* %r for the replication factor
* %u for owner's user name
* %x and %X for access data
* %y and %Y for modification date

You can combine multiple format options in the same command as follows:

```
hdfs dfs -stat -R "%n, %u, %g, %x, %y, %a, %b, %o, %r" /user/<username>/myfolder
```

This command prints a set of statistics in a comma seperated value format as in:
```
myfolder, ubuntu, supergroup, 1970-01-01 00:00:00, 2018-04-24 21:43:59, 755, 0, 0, 0
LICENSE.txt, ubuntu, supergroup, 2018-04-24 21:43:57, 2018-04-24 21:43:58, 644, 147145, 134217728, 1
NOTICE.txt, ubuntu, supergroup, 2018-04-24 21:43:58, 2018-04-24 21:43:59, 644, 21867, 134217728, 1
README.txt, ubuntu, supergroup, 2018-04-24 21:43:59, 2018-04-24 21:43:59, 644, 1366, 134217728, 1
test.txt, ubuntu, supergroup, 2018-04-24 21:43:59, 2018-04-24 21:43:59, 644, 0, 134217728, 1
```

**Copy files from HDFS**

```
$ hdfs dfs -get /user/<username>/myfolder/test.txt ./test.txt
```

Copies the file from HDFS to the local filesystem. If the destination path is not provided, a local file with the same name as the source will be created.

When copying multiple files from HDFS to the local filesystem, the destination must be the destination folder as in:

```
$ hdfs dfs -get /user/<username>/myfolder/*.txt <mylocalfolder>
```

**Copying and moving files in HDFS**

```
$ hdfs dfs -cp myfolder/test.txt myfolder/newtest.txt
```

Copies the file to the destination in HDFS.

As in copying to the local filesystem, when copying multiple files, the destination must be a folder as in:

```
$ hdfs dfs -mkdir newfolder
$ hdfs dfs -cp myfolder/*.txt newfolder/
```

Moving files in HDFS from one place to another is similar to copying files.

```
$ hdfs dfs -mv myfolder/newtest.txt myfolder/test2.txt
```

Following this command, `myfolder/newtest.txt` will not exist anymore. It is now `myfolder/test2.txt`.

**Deleting files and directories**

```
$ hdfs dfs -rm `myfolder/test2.txt`
``` 

Deletes the file names `myfolder/test2.txt` from HDFS. This command is equivalent to the Unix command `rm <src>`.

You can delete multiple files that respect a given pattern as in:

```
$ hdfs dfs -rm `myfolder/*.txt`
``` 

To delete files and directories recursively, you can use:

```
$ hdfs dfs -rm -r `newfolder/`
```

You can also use `rmdir` command to remove an empty directory  from HDFS. 

```
$ hdfs dfs -rmdir -r `newfolder/`
```

**Check HDFS disk usage**

You can check HDFS filesystem disk space usage with:

```
hdfs dfs -df
```

**Check HDFS health**

HDFS is an essential component in a Hadoop Big data cluster, ensuring its health is critical.
You can check for HDFS file system health with:

```
hdfs fsck /
```

You can print HDFS block report with:

```
hdfs fsck / -files -blocks
```

You can list corrupt blocks with:

```
hdfs fsck -list-corruptfileblocks
```

**HDFS `safemode`**

HDFS `safemode` is a maintenance state of the `NameNode`, during which it doesn’t allow any modifications to HDFS file system. In this state, HDFS is in `read-only` mode and does not replicate or delete Data Blocks.

The name node anters in safe mode temporarily at startup. The cluster is also configured to enable automatic safe
mode when:

* the name node is full 
* the percentage of blocks with minimal replication requirement is less than the configured threshold (0.999 by default).
* the number of alive data nodes is less than the configured threshold

To check the status of safe mode:

```
hdfs dfsadmin –safemode get
```

To enter in safe mode:

```
hdfs dfsadmin –safemode enter
```

While in safe mode, try to write to HDFS by creating a folder, adding a file or deleting a file and see how
your request will be rejected.

To leave safe mode:

```
hdfs dfsadmin –safemode leave
```

**Stop the Namenode**

You can stop the `NameNode` and `DataNode` deamon as follows:

```
$ ./sbin/stop-dfs.sh
```
