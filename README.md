
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

 <property>
    <name>javax.jdo.option.ConnectionURL</name>
    <value>jdbc:mysql://my.mapr01.fr:3306/hive?createDatabaseIfNotExist=true</value>
    <description>JDBC connect string for a JDBC metastore</description>
</property>

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

-----------------------------

## MYSQL MASTER/MASTER Configuration 

How to configure 2 servers in a Master/Master configuration. In the HAProxy config file we define the second server as "backup" to avoid any issues between different tools using the HCat. So the first server is the default server and the second server that is also a master is only used by HAProxy when the first one is really unvailable. 

Configuration to made in the two servers : 

            First MySQL Server (default) | Second MySQL Server (backup)
            ------------ | -------------
            /etc/my.cnf :       | /etc/my.cnf :
                    | Content in the second column

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



Refs used : 

How To Set Up Master Slave Replication in MySQL :
https://www.digitalocean.com/community/tutorials/how-to-set-up-master-slave-replication-in-mysql

How To Use HAProxy to Set Up MySQL Load Balancing : 
https://www.digitalocean.com/community/tutorials/how-to-use-haproxy-to-set-up-mysql-load-balancing--3








