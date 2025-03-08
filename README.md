# PostgreSQL-Cluster-Setup

**In this, I will take you through setting up a PostgreSQL primary and secondary node cluster for high availability and failover. PostgreSQL is a powerful, open-source relational database known for its reliability, robustness, and extensibility. In this scenario, we will automate the process of setting up a primary database node and one or more secondary nodes using streaming replication. This setup ensures that all write operations occur on the primary node, while read operations can be distributed across the secondary nodes to improve performance and scalability. In case of failure, the secondary node can be promoted to primary, ensuring minimal downtime and high availability. This solution is essential for mission-critical applications that require fault tolerance and data redundancy in a cloud-native environment.**
  
<h2>Step 1: Setup the Primary PostgreSQL node.</h2>
To manually configure the Apt repository, follow these steps:

    sudo apt install curl ca-certificates
    sudo install -d /usr/share/postgresql-common/pgdg
    sudo curl -o /usr/share/postgresql-common/pgdg/apt.postgresql.org.asc --fail https://www.postgresql.org/media/keys/ACCC4CF8.asc
    sudo sh -c 'echo "deb [signed-by=/usr/share/postgresql-common/pgdg/apt.postgresql.org.asc] https://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'
    sudo apt update
    sudo apt -y install postgresql
<h2>Step 2: Configure the Primary PostgreSQL node.</h2>
Modify the configurations of the primary node as follows:
    
    sudo sed -i "s@#listen_addresses = 'localhost'@listen_addresses = '*'@g" /etc/postgresql/15/main/postgresql.conf
    #wal_level = replica
    sudo sed -i "s@#wal_level = replica@wal_level = replica@g" /etc/postgresql/15/main/postgresql.conf
    # max_wal_senders: the maximum number of WAL sender processes.
    sudo sed -i "s@#max_wal_senders = 10@max_wal_senders = 10@g" /etc/postgresql/15/main/postgresql.conf
    # wal_keep_size: the minimum size of past WAL files kept on the primary node.
    sudo sed -i "s@#wal_keep_size = 0@wal_keep_size = 128MB@g" /etc/postgresql/15/main/postgresql.conf
    # # archive_mode: whether to enable WAL archiving. Set the value to on.
    sudo sed -i "s@#archive_mode = off@archive_mode = on@g" /etc/postgresql/15/main/postgresql.conf

<h2>Step 2: Modify the pg_hba.conf file to configure the permissions for the secondary node to connect to the primary node.</h2>

- Replace <YOUR_USER> with a username of the secondary node.
- Replace <Private IP address or CIDR block of the secondary node> with the private IP address of the ECS instance used as the secondary node or the CIDR block to which the private IP address belongs.

    sudo -u postgres psql -c "CREATE ROLE <YOUR_USER> REPLICATION LOGIN PASSWORD '<YOUR_PASSWORD>';"
