
# How to use haproxy with a MapR cluster HA and a MySQL MASTER/MASTER configuration


## Hostname / DNS / hosts file 
You have to create a name in your DNS or hosts file.

    # PRODUCTION CLUSTER
        10.68.7.91  ip-10-68-7-91		node1
        10.68.7.92  ip-10-68-7-92   	node2
        10.68.7.93  ip-10-68-7-93    	node3
        10.68.7.94  ip-10-68-7-94    	node4
        10.68.7.95  ip-10-68-7-95    	node5
        
    # LOAD BALANCER / VPN / PROXY
        10.68.7.31  ip-10-68-7-31    	node0
        10.68.7.31  my.mapr01.fr		prod1
        

You can now use the cluster my.mapr01.fr in you differen configuration files in the MapR Cluster.
Don't forget to replicate your configurations files in the different nodes running the sames services.
We can use cluster shell to replicate files in the cluster. 

-----------------------------
## Hive Configuration 

You can find bellow the 2 section the /opt/mapr/hive/hive-1.2/conf/hive-site.xml that you have to update.

         <property>
            <name>javax.jdo.option.ConnectionURL</name>
            <value>jdbc:mysql://my.mapr01.fr:3306/hive?createDatabaseIfNotExist=true</value>
            <description>JDBC connect string for a JDBC metastore</description>
        </property>
... 

         <property>
            <name>hive.metastore.uris</name>
            <value>thrift://my.mapr01.fr:9083</value>
         </property>

You can distribute this file to nodes that will have hive installed using clustershell :

        [root@ip-10-68-7-91 conf]# clush -a --copy /opt/mapr/hive/hive-1.2/conf/hive-site.xml
        node4: scp: /opt/mapr/hive/hive-1.2/conf/: Is a directory
        node3: scp: /opt/mapr/hive/hive-1.2/conf/: Is a directory
        node5: scp: /opt/mapr/hive/hive-1.2/conf/: Is a directory
        clush: node4: exited with exit code 1
        clush: node3: exited with exit code 1
        clush: node5: exited with exit code 1

Nodes 3 to 5 don't have hive installed so cannot copy this file in this hive conf directory, no worry. :wink:

:warning:  If you have also Spark installed on your cluster don't forget to copy hive-site.xml also in the Spark conf folder.

        cp /opt/mapr/hive/hive-1.2/conf/hive-site.xml /opt/mapr/spark/spark-1.5.2/conf

-----------------------------
## HUE Configuration 

To configure Hue you have to update this file : /opt/mapr/hue/hue-3.9.0/desktop/conf/hue.ini

Sections where host must be updated : 

    [[database]]
        engine=mysql
        ######## host=ip-10-68-7-91
        host=my.mapr01.fr
        port=3306

    [hadoop]
        [[hdfs_clusters]]
            [[[default]]]
                ######## webhdfs_url=http://ip-10-68-7-91:14000/webhdfs/v1
                webhdfs_url=http://my.mapr01.fr:14000/webhdfs/v1


:warning:
For the Yarn Cluster section we don't need to manage HA of the 3 RM in the cluster. MapR already manage the failover with Zookeeper. We have to mention the virtual name for the History Server 



    [[yarn_clusters]]
        [[[default]]]
            # Enter the host on which you are running the ResourceManager
            #resourcemanager_host=ip-10-68-7-29
            resourcemanager_port=8032
            submit_to=True
            security_enabled=${security_enabled}
            mechanism=${mechanism}

            # URL of the HistoryServer API
            ######### history_server_api_url=http://ip-10-68-7-91:19888
            history_server_api_url=http://my.mapr01.fr:19888


    [liboozie]
        # The URL where the Oozie service runs on. This is required in order for
        # users to submit jobs.
        ######## oozie_url=http://ip-10-68-7-91:11000/oozie
        oozie_url=http://my.mapr01.fr:11000/oozie


    [beeswax]
        ######## hive_server_host=ip-10-68-7-91
        hive_server_host=my.mapr01.fr


    [sqoop]
        # For autocompletion, fill out the librdbms section.
        # Sqoop server URL
        ######## server_url=http://ip-10-68-7-91:12000/sqoop
        server_url=http://my.mapr01.fr:12000/sqoop


    [hbase]
        ######## hbase_clusters=(Cluster|ip-10-68-7-91:9090)
        hbase_clusters=(Cluster|my.mapr01.fr:9090)


    [spark]
        # Host address of the Livy Server.
        ######## livy_server_host=ip-10-68-7-91
        livy_server_host=my.mapr01.fr


Like for hive you can also distribute this file to nodes that will have hive installed using clustershell :

            [root@ip-10-68-7-91 ~]# clush -a --copy /opt/mapr/hue/hue-3.9.0/desktop/conf/hue.ini
            node5: scp: /opt/mapr/hue/hue-3.9.0/desktop/conf/: No such file or directory
            node4: scp: /opt/mapr/hue/hue-3.9.0/desktop/conf/: No such file or directory
            clush: node5: exited with exit code 1
            node3: scp: /opt/mapr/hue/hue-3.9.0/desktop/conf/: No such file or directory
            clush: node4: exited with exit code 1
            clush: node3: exited with exit code 1
            [root@ip-10-68-7-91 ~]#

Nodes 3 to 5 don't have hive installed so cannot copy this file in this hive conf directory, no worry. :wink:

----------------------------

## Config update for the Job History server 

Update the file /opt/mapr/hadoop/hadoop-2.7.0/etc/hadoop/mapred-site.xml 
Change localhost by the HAProxy host name. There is a rule in HAproxy to get the UI on port 19888 in HA. 

          <property>
            <name>mapreduce.jobhistory.address</name>
            <value>my.mapr01.fr:10020</value>
          </property>
          <property>
            <name>mapreduce.jobhistory.webapp.address</name>
            <value>my.mapr01.fr:19888</value>
          </property>

-----------------------------

## MYSQL MASTER/MASTER Configuration 

How to configure 2 servers in a Master/Master configuration. In the HAProxy config file we define the second server as "backup" to avoid any issues between different tools using the HCat. So the first server is the default server and the second server that is also a master is only used by HAProxy when the first one is really unvailable. 

Configuration to made in the two servers : 

First MySQL Server (default) :

/etc/my.cnf :

            [mysqld]   |
            datadir=/var/lib/mysql   |
            socket=/var/lib/mysql/mysql.sock   |
            user=mysql   |
            # Disabling symbolic-links is recommended to prevent assorted security risks   |
            symbolic-links=0   |
            server-id=1   | 
            log_bin=/var/log/mysql/mysql-bin.log    |
            binlog_do_db=hive  |
            binlog_do_db=hue   |
            binlog_do_db=oozie   |
            binlog_format=row   |
            
            [mysqld_safe]   |
            log-error=/var/log/mysqld.log   |
            pid-file=/var/run/mysqld/mysqld.pid   |

Second MySQL Server (backup)


Refs used : 

How To Set Up Master Slave Replication in MySQL :
https://www.digitalocean.com/community/tutorials/how-to-set-up-master-slave-replication-in-mysql

How To Use HAProxy to Set Up MySQL Load Balancing : 
https://www.digitalocean.com/community/tutorials/how-to-use-haproxy-to-set-up-mysql-load-balancing--3








