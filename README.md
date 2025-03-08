# PostgreSQL-Cluster-Setup

![](/images/postgresql-12-streaming-replication.jpg)

**In this, I will take you through setting up a PostgreSQL primary and secondary node cluster for high availability and failover. PostgreSQL is a powerful, open-source relational database known for its reliability, robustness, and extensibility. In this scenario, we will automate the process of setting up a primary database node and one or more secondary nodes using streaming replication. This setup ensures that all write operations occur on the primary node, while read operations can be distributed across the secondary nodes to improve performance and scalability. In case of failure, the secondary node can be promoted to primary, ensuring minimal downtime and high availability. This solution is essential for mission-critical applications that require fault tolerance and data redundancy in a cloud-native environment.**

<h1>Setup the Primary PostgreSQL node.</h1>
<h2>Step 1: Manually configure the Apt repository.</h2>

    sudo apt install curl ca-certificates
    sudo install -d /usr/share/postgresql-common/pgdg
    sudo curl -o /usr/share/postgresql-common/pgdg/apt.postgresql.org.asc --fail https://www.postgresql.org/media/keys/ACCC4CF8.asc
    sudo sh -c 'echo "deb [signed-by=/usr/share/postgresql-common/pgdg/apt.postgresql.org.asc] https://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'
    sudo apt update
    sudo apt -y install postgresql
    
<h2>Step 2: Configure the Primary PostgreSQL node.</h2>
Modify the configurations of the primary node as follows:
    
    sudo sed -i "s@#listen_addresses = 'localhost'@listen_addresses = '*'@g" /etc/postgresql/17/main/postgresql.conf
    #wal_level = replica
    sudo sed -i "s@#wal_level = replica@wal_level = replica@g" /etc/postgresql/17/main/postgresql.conf
    # max_wal_senders: the maximum number of WAL sender processes.
    sudo sed -i "s@#max_wal_senders = 10@max_wal_senders = 10@g" /etc/postgresql/17/main/postgresql.conf
    # wal_keep_size: the minimum size of past WAL files kept on the primary node.
    sudo sed -i "s@#wal_keep_size = 0@wal_keep_size = 128MB@g" /etc/postgresql/17/main/postgresql.conf
    # # archive_mode: whether to enable WAL archiving. Set the value to on.
    sudo sed -i "s@#archive_mode = off@archive_mode = on@g" /etc/postgresql/17    /main/postgresql.conf

<h2>Step 3: Modify the pg_hba.conf file to configure the permissions for the secondary node to connect to the primary node.</h2>

- Replace <YOUR_USER> with a username of the secondary node.
- Replace <Private IP address or CIDR block of the secondary node> with the private IP address of the ECS instance used as the secondary node or the CIDR block to which the private IP address belongs.

        sudo -u postgres psql -c "CREATE ROLE <YOUR_USER> REPLICATION LOGIN PASSWORD '<YOUR_PASSWORD>';"

<h2>Step 4: Restart PostgreSQL and enable PostgreSQL to start on system startup.</h2>

    sudo systemctl restart postgresql.service
    sudo systemctl enable postgresql.service

<h1>Setup the Secondary PostgreSQL node.</h1>

<h2>Step 1: Manually configure the Apt repository.</h2>

    sudo apt install curl ca-certificates
    sudo install -d /usr/share/postgresql-common/pgdg
    sudo curl -o /usr/share/postgresql-common/pgdg/apt.postgresql.org.asc --fail https://www.postgresql.org/media/keys/ACCC4CF8.asc
    sudo sh -c 'echo "deb [signed-by=/usr/share/postgresql-common/pgdg/apt.postgresql.org.asc] https://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'
    sudo apt update
    sudo apt -y install postgresql

<h2>Step 2: Stop PostgreSQL.</h2>

    sudo systemctl stop postgresql.service

<h2>Step 3: Delete the initialization data of the secondary node.</h2>

    sudo rm -rf /var/lib/postgresql/17/main/

<h2>Step 4: Use the pg_basebackup utility to create a base backup of the primary node on the secondary node.</h2>

- Replace <YOUR_USER> with a username of the primary node.
- Replace <YOUR_PASSWORD> with the password of the primary node.
- Replace <Private IP address of the primary node> with the private IP address of the ECS instance that is used as the primary node.

        cd /
        export PGPASSWORD=<YOUR_PASSWORD>
        sudo -E -u postgres pg_basebackup -h <Private IP address of the primary node> -D /var/lib/postgresql/17/main/ -U <YOUR_USER> -P -w -v --wal-method=stream

<h2>Step 5: Modify the postgresql.conf file of the secondary node.</h2>

    # hot_standby: whether to enable read-only queries on the secondary node. Set the value to on.
    sudo sed -i "s@#hot_standby = off@hot_standby = on@g" /etc/postgresql/17/main/postgresql.conf
    # hot_standby_feedback: whether to allow the secondary node to send feedback about replication status and progress to the primary node. Set the value to on.
    sudo sed -i "s@#hot_standby_feedback = off@hot_standby_feedback = on@g" /etc/postgresql/17/main/postgresql.conf

<h2>Step 6: Configure the connection information of the primary node.</h2>

- Replace <Private IP address of the primary node> with the private IP address of the ECS instance that is used as the primary node.
- Replace <YOUR_USER> with a username of the primary node.
- Replace <YOUR_PASSWORD> with the password of the primary node.

        sudo sed -i "s@#primary_conninfo = ''@primary_conninfo = 'host=<Private IP address of the primary node> port=5432 user=<YOUR_USER> password=<YOUR_PASSWORD>'@g" /etc/postgresql/15/main/postgresql.conf

<h2>Step 7: Configure the secondary node to be able to take over from the primary node.</h2>

    sudo -u postgres touch /var/lib/postgresql/17/main/standby.signal

<h2>Step 8: Restart PostgreSQL and enable PostgreSQL to start on system startup.</h2>

    sudo systemctl restart postgresql.service
    sudo systemctl enable postgresql.service

<h1>Test the primary/secondary PostgreSQL architecture.</h1>

    sudo pg_basebackup -D /var/lib/pgsql/17/data -h <IP address of the primary node> -p 5432 -U replica -X stream -P

<h2> Step 1: Run the following command on the primary node to check whether the sender process is available.</h2>

    ps aux |grep sender

- The following command output indicates that the sender process is available:

      postgres  2916  0.0  0.3 340388  3220 ?        Ss   15:38   0:00 postgres: walsender  replica 192.168.**.**(49640) streaming 0/F01C1A8

<h2>Step 2: Run the following command on the secondary node to check whether the receiver process is available.</h2>

    ps aux |grep receiver

- The following command output indicates that the receiver process is available:

        postgres 23284  0.0  0.3 387100  3444 ?        Ss   16:04   0:00 postgres: walreceiver   streaming 0/F01C1A8

<h2>Step 3: On the primary node, access the PostgreSQL interactive terminal and execute an SQL statement to check the status of the secondary node.</h2>

- Run the following command to log on to PostgreSQL by using the postgres account:

        sudo su - postgres

- Run the following command to access the PostgreSQL interactive terminal:

        psql
  
- Execute the following statement to check the status of the secondary node:

        select * from pg_stat_replication;
  
- The following result indicates the status of the secondary node:

        pid | usesysid | usename | application_name | client_addr | client_hostname | client_port | backend_start | backend_xmin | state | sent_location | write_locati
        on | flush_location | replay_location | sync_priority | sync_state 
        ------+----------+---------+------------------+---------------+-----------------+------------- +-------------------------------+--------------+-----------+---------------+-------------
        ---+----------------+-----------------+---------------+------------
        2916 | 16393 | replica | walreceiver | 192.168.**.** | | 49640 | 2017-05-02 15:38:06.188988+08 | 1836 | streaming | 0/F01C0C8 | 0/F01C0C8 
        | 0/F01C0C8 | 0/F01C0C8 | 0 | async
        (1 rows)

- Run the following command and press the Enter key to exit the PostgreSQL interactive terminal:

        \q
  
- Run the following command and press the Enter key to exit PostgreSQL:

        exit


<h1>Conclusion</h1>
By following this guide, you have successfully set up a PostgreSQL primary and secondary node cluster using streaming replication. This ensures high availability, improved performance, and minimal downtime in case of a primary node failure. Happy database clustering!

 

    
