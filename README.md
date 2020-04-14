# Bundle: db

The `db` bundle is preconfigured to synchronize Hazelcast with MySQL running on `localhost`. It includes the `db` cluster and `perf_test_db` app to read/write from/to Hazelcast and MySQL. It also includes instructions for replacing MySQL with another database.

## Installing Bundle

```console
install_bundle -download bundle-hazelcast-3-app-perf_test_db-cluster-db
```

## Use Case

The client applications read/write from/to the Hazelcast which in turn read/write from/to a database. The database is used as the persistent store and Hazelcast as the bidirectional cache-aside store. The Hazelcast maps are configured with the LFU eviction policy to evict entries if the free heap size falls below 25% of the maximum heap size. This ensures the Hazelcast cluster will have at least 25% of free memory at all time which is necessary for executing distributed operations such as query and task executions.

![DB Sync Screenshot](/images/db-sync.png)

## Configuring MySQL

The 'db' cluster has been preconfigured to connect to MySQL on localhost with the user name `root` and the password `MySql123`. You can change the user name and password in `etc/hibernate.cfg-mysql.xml`.

*If you have a database other than MySQL than follow the instructions in the [Replacing MySQL with Another Database](#replacing-mysql-with-another-database) section.*

```console
switch_cluster db
vi etc/hibernate.cfg-mysql.xml
```

### `nw` Schema

Open MySQL Workbench and create the `nw` schema. When you run the `test_group` script (see below), the `customers` and `orders` tables will automatically be created by Hibernate invoked by the `MapStorePkDbImpl` plugin.

*If you have a database other than MySQL, then make sure to create the `nw` schema using the appropriate command.*

## Building `perf_test_db`

You need to download the MySQL binary files by building `perf_test_db` as follows.

```console
cd_app perf_test_db; cd bin_sh
./build_app
```

## Running Cluster

First, add at least two (2) members to the `db` cluster. All bundles come without members.

```console
add_member; add_member
```

Run the cluster.

```console
start_cluster

# Monitor the log file. Hibernate has been configured to log SQL statements
# executed by the MapStorePkDbImpl plugin.
show_log
```

## Ingesting Data

The `test_group` script creates mock data for `Customer` and `Order` objects and ingests them into the Hazelcast cluster which in turn writes to MySQL via the `MapStorePkDbImpl` plugin included in the PadoGrid distribution. The same plugin is also registered to retrieve data from MySQL for cache misses in the cluster.

```console
cd_app perf_test_db; cd bin_sh
./test_group -prop ../etc/group-factory.properties  -run
```

You should see SQL statements being logged if you are running `show_log`.

## Replacing MySQL with Another Database

If you have a database other than MySQL then you need to download the appropriate JDBC driver and update the Hibernate configuration file. The JDBC driver can be downloaded by adding the database dependency in the `pom.xml` file in the `perf_test_db` directory and run the `build_app` command as shown below.

```console
cd_app perf_test_db
vi pom.xml
```

The following shows the PostgresSQL dependency that is already included in the `pom.xml` file.

```xml
<!-- https://mvnrepository.com/artifact/org.postgresql/postgresql -->
<dependency>
   <groupId>org.postgresql</groupId>
   <artifactId>postgresql</artifactId>
   <version>42.2.8</version>
</dependency>
```

Build the app. The `build_app` command downloads your JDBC driver and automatically deploys it to the `$PADOGRID_WORKSPACE/lib` directory.

```console
cd bin_sh
./build_app
```

Configure Hibernate. Copy the `etc/hibernate.cfg-mysql.xml` file in the cluster directory to a file with another name, e.g., `hibernate.cfg-mydb.xml` and include the driver and login information in the new file.

```console
cd_cluster
cp etc/hibernate.cfg-mysql.xml etc/hibernate.cfg-mydb.xml
vi etc/hibernate.cfg-mydb.xml
```

Set the cluster to the new Hibernate configuration file. Edit `bin_sh/setenv.sh` and set the `HIBERNATE_CONFIG_FILE` environment variable to the new Hibernate configuration file name.

```console
cd_cluster
vi bin_sh/setenv.sh
```

For our example, you would set `HIBERNATE_CONFIG_FILE` in the `bin_sh/setenv.sh` file as follows:

```bash
HIBERNATE_CONFIG_FILE="$CLUSTER_DIR/etc/hibernate.cfg-mydb.xml"
```

Go to the [`nw` Schema](#nw-schema) section and continue.

## Tearing Down

```console
# Stop the cluster.
stop_cluster
```
