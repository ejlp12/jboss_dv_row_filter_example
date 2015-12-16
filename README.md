# Row filter example:


Start database, create table and insert sample data:


```
H2_DATA_DIR=/Servers/h2-data
JDV_HOME=/Servers/JDV-6.2

mkdir -p /Servers/h2-data

cat > starth2.sh << __EOF__
JDV_HOME=$JDV_HOME
H2_DATA_DIR=$H2_DATA_DIR
java -cp "\$JDV_HOME/modules/system/layers/base/com/h2database/h2/main/h2-1.3.168.redhat-4.jar:$CLASSPATH" org.h2.tools.Server -tcp -tcpPort 9091 -baseDir \$H2_DATA_DIR -web -webPort 8083 &
__EOF__

chmod 755 starth2.sh


cat > initdata.sql << __EOF__
CREATE TABLE IF NOT EXISTS PERSON(id int identity primary key, name varchar(255), groupname varchar(255));

INSERT INTO PERSON(id, name, groupname) VALUES(1, 'Unyil', 'A');
INSERT INTO PERSON(id, name, groupname) VALUES(2, 'Kinoy', 'A');
INSERT INTO PERSON(id, name, groupname) VALUES(3, 'Cuplis', 'B');
INSERT INTO PERSON(id, name, groupname) VALUES(4, 'Usro', 'B');

CREATE TABLE IF NOT EXISTS GROUPCITYMAP(id int identity primary key, groupname varchar(255),  city varchar(255));

INSERT INTO GROUPCITYMAP(id, groupname, city) VALUES(1, 'A', 'Bandung');
INSERT INTO GROUPCITYMAP(id, groupname, city) VALUES(2, 'A', 'Cianjur');
INSERT INTO GROUPCITYMAP(id, groupname, city) VALUES(3, 'A', 'Sukabumi');
INSERT INTO GROUPCITYMAP(id, groupname, city) VALUES(4, 'B', 'Semarang');
INSERT INTO GROUPCITYMAP(id, groupname, city) VALUES(5, 'B', 'Purwokerto');
INSERT INTO GROUPCITYMAP(id, groupname, city) VALUES(6, 'B', 'Solo');
__EOF__

java -cp "$JDV_HOME/modules/system/layers/base/com/h2database/h2/main/h2-1.3.168.redhat-4.jar:$CLASSPATH" org.h2.tools.RunScript -url jdbc:h2:tcp://localhost:9091/Servers/h2-data -script initdata.sql

java -cp "$JDV_HOME/modules/system/layers/base/com/h2database/h2/main/h2-1.3.168.redhat-4.jar:$CLASSPATH" org.h2.tools.Shell -url jdbc:h2:tcp://localhost:9091/Servers/POC_PAJAK/h2-data -user sa -password "" -sql "SELECT * from PERSON; SELECT * FROM GROUPCITYMAP;"
```


Create datasource:

```
$JDV_HOME/bin/jboss-cli.sh -c --command="/subsystem=datasources/data-source=poc_row_filter-ds:add(jndi-name=java:/poc_row_filter, driver-name=h2, connection-url=jjdbc:h2:tcp://localhost:9091/Servers/h2-data, user-name=sa, password=\"\")"
$JDV_HOME/bin/jboss-cli.sh -c --command="/subsystem=datasources/data-source=poc_row_filter-ds:enable"
```

Above script will create following configuration in `standalone.xml`:

```xml
    <datasource jndi-name="java:/poc_row_filter" pool-name="poc_row_filter" enabled="true">
        <connection-url>jdbc:h2:tcp://localhost:9091/Servers/h2-data</connection-url>
        <driver>h2</driver>
        <security>
            <user-name>sa</user-name>
            <password></password>
        </security>
    </datasource>
```

Add user:

```
$JDV_HOME/bin/add-user.sh -a  -u unyil -p Passw0rd! -g user
$JDV_HOME/bin/add-user.sh -a  -u usro -p Passw0rd! -g user
```

```
cd $JDV_HOME/dataVirtualization/teiid-adminshell
unzip teiid-8.7.1.6_2-redhat-6-adminshell-dist.zip
cd teiid-adminshell-8.7.1.6_2-redhat-6/
```

Deploy VDB:

```
$JDV_HOME/bin/add-user.sh  -u demoAdmin -p Passw0rd! -g admin,user
./adminshell.sh -Dvdbname=poc_row_filter -Dvdbfile=/Users/ejlp12/workspace_teiid/poc_row_filter/poc_row_filter.vdb .  ./examples/DeployAndVerifyVDB.groovy
```

Test using adminshell:



```
cd $JDV_HOME/dataVirtualization/teiid-adminshell/teiid-adminshell-8.7.1.6_2-redhat-6/
./adminshell.sh

groovy:000> sql = connect("jdbc:teiid:poc_row_filter@mm://localhost:31000", "unyil", "Passw0rd!");
===> groovy.sql.TeiidSql@77cf3f8b
groovy:000> sql.eachRow("select * from sample.groupcitymap") { println "${it}" }
[ID:1, GROUPNAME:A, CITY:Bandung]
[ID:2, GROUPNAME:A, CITY:Cianjur]
[ID:3, GROUPNAME:A, CITY:Sukabumi]
===> null

groovy:000> sql = connect("jdbc:teiid:poc_row_filter@mm://localhost:31000", "usro", "Passw0rd!");
===> groovy.sql.TeiidSql@2ed3b1f5
groovy:000> sql.eachRow("select * from sample.groupcitymap") { println "${it}" }
[ID:4, GROUPNAME:B, CITY:Semarang]
[ID:5, GROUPNAME:B, CITY:Purwokerto]
[ID:6, GROUPNAME:B, CITY:Solo]
===> null
```