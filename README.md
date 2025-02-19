# PostgreSQL-Cluster-Setup

**In this, I will take you through setting up a PostgreSQL primary and secondary node cluster for high availability and failover. PostgreSQL is a powerful, open-source relational database known for its reliability, robustness, and extensibility. In this scenario, we will automate the process of setting up a primary database node and one or more secondary nodes using streaming replication. This setup ensures that all write operations occur on the primary node, while read operations can be distributed across the secondary nodes to improve performance and scalability. In case of failure, the secondary node can be promoted to primary, ensuring minimal downtime and high availability. This solution is essential for mission-critical applications that require fault tolerance and data redundancy in a cloud-native environment.**
  
<h2>Step 1: Setup the primary PostgreSQL node.</h2>

