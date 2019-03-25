##  How To Create Oracle DB Dataguard in K8S
### Requirement
   We run critical workload in Oracle DB. We have dataguard Implementation on-premise.
   We plan to move DB to OKE (Oracle Kubernetes Engine), we would like to maintain the same high availability DB.
   So we plan to create dataguard in k8s via statefulset

#### Create Docker images of DB and Move into K8S
Please refer details on other [notes](https://www.henryxieblogs.com/search?q=db+to+k8s)
#### Create yaml files for Primary and Standby
* PV and PVC Sample yaml

```
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: livesqlstg-pv-nfs-volume1
spec:
  capacity:
    storage: 1Ti
  accessModes:
    - ReadWriteMany
  storageClassName: ocifss
  nfs:
    path: "/livesqlstg-fss"
    server: 100.100.201.224
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: livesqlstg-pv-nfs-claim1
spec:
  storageClassName: ocifss
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Ti
```
* Primary DB yaml file example:
```
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: livesqlsb-db
  labels:
    name: livesqlsb-db-service
spec:
  selector:
        matchLabels:
           name: livesqlsb-db-service
  serviceName: livesqlsb-db-service
  replicas: 1
  template:
    metadata:
        labels:
           name: livesqlsb-db-service
    spec:
      securityContext:
         runAsUser: 54321
         fsGroup: 54321
      volumes:
        - name: livesqlstg-pv-nfs-volume1
          persistentVolumeClaim:
            claimName: livesqlstg-pv-nfs-claim1
      containers:
        - image: oracle/database:19.2v2
          name: livesqldb
          ports:
            - containerPort: 1521
              name: livesqldb
          volumeMounts:
            - mountPath: /opt/oracle/oradata
              name: livesqlsb-db-pv-storage
            - mountPath: /livesqlstg-fss
              name: livesqlstg-pv-nfs-volume1
          env:
            - name: ORACLE_SID
              value: "ALSSTG"
            - name: ORACLE_PDB
              value: "testpdb"
      imagePullSecrets:
      - name: iad-ocir-secret
      nodeSelector:
             dbhost: livesqlsb
  volumeClaimTemplates:
  - metadata:
      name: livesqlsb-db-pv-storage
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 1024Gi
```
* Standby DB yaml file example:
 ```
 apiVersion: apps/v1
 kind: StatefulSet
 metadata:
   name: livesqlsb-db-dr
   labels:
     name: livesqlsb-db-dr-service
 spec:
   selector:
         matchLabels:
            name: livesqlsb-db-dr-service
   serviceName: livesqlsb-db-service
   replicas: 1
   template:
     metadata:
         labels:
            name: livesqlsb-db-dr-service
     spec:
       securityContext:
          runAsUser: 54321
          fsGroup: 54321
       volumes:
         - name: livesqlstg-pv-nfs-volume1
           persistentVolumeClaim:
             claimName: livesqlstg-pv-nfs-claim1
       containers:
         - image: iad.ocir.io/espsnonprodint/livesqlsandbox/oracle-database:19.2v2
           name: livesqldb
           ports:
             - containerPort: 1521
               name: livesqldb
           volumeMounts:
             - mountPath: /opt/oracle/oradata
               name: livesqlsb-db-pv-storage
             - mountPath: /livesqlstg-fss
               name: livesqlstg-pv-nfs-volume1
           env:
             - name: ORACLE_SID
               value: "ALSSTGDR"
             - name: ORACLE_PDB
               value: "testpdb"
       imagePullSecrets:
       - name: iad-ocir-secret
       nodeSelector:
              dbhost: livesqlsb
   volumeClaimTemplates:
   - metadata:
       name: livesqlsb-db-pv-storage
     spec:
       accessModes: [ "ReadWriteOnce" ]
       resources:
         requests:
           storage: 1024Gi
 ```
 * Start both primary and standby via yaml files
 * A brand new db will be created for standby, then we will convert it to physical Standby
 #### Convert brand new DB to Standby
 * Change DB to archivelog mode
   * SQL>alter system set db_recovery_file_dest='/livesqlstg-fss/oracle' scope=both;
   * SQL>alter system set db_recovery_file_dest_size=1000G  scope=both;
   * SQL>shutdown immediate
   * SQL>startup mount
   * SQL>alter database archivelog;
   * SQL>alter database open;
 * Change db_name of the new DB
   * sql> create pfile='/opt/oracle/oradata/dbconfig/ALSSTGDR/pfileALSSTGDR.ora' from spfile;
   * Update pfileALSSTGDR.ora
     * Change db_name to be same as primary ie 'ALSSTG'
     * Add *.db_unique_name to be ie 'ALSSTGDR'
     * Add *.db_file_name_convert='ALSSTG','ALSSTGDR'
 * Create K8S service names for primary and standby dbs
   * Primary yaml file is like

```
apiVersion: v1
kind: Service
metadata:
  labels:
     name: livesqlsb-db-service
  name: livesqlsb-db-service
  namespace: default
spec:
  ports:
  - port: 1521
    protocol: TCP
    targetPort: 1521
  selector:
     name: livesqlsb-db-service
---
apiVersion: v1
kind: Service
metadata:
  labels:
     name: livesqlsb-db-dr-service
  name: livesqlsb-db-dr-service
  namespace: default
spec:
  ports:
  - port: 1521
    protocol: TCP
    targetPort: 1521
  selector:
     name: livesqlsb-db-dr-service
```
 * Create TNSNAMES ALSSTGPRI (primary) ALSSTGSTB (standby) via add below entry. tnsping alsstgpri alsstgstb to verify

```
ALSSTGPRI=
(DESCRIPTION =
  (ADDRESS = (PROTOCOL = TCP)(HOST = livesqlsb-db-service)(PORT = 1521))
  (CONNECT_DATA =
    (SERVER = DEDICATED)
    (SERVICE_NAME = ALSSTG)
  )
)

ALSSTGSTB=
(DESCRIPTION =
 (ADDRESS = (PROTOCOL = TCP)(HOST = livesqlsb-db-dr-service)(PORT = 1521))
 (CONNECT_DATA =
   (SERVER = DEDICATED)
   (SERVICE_NAME = ALSSTGDR)
 )
)
```
 * Add below into listener.ora on standby side to manually register DB into listener
 ```
 SID_LIST_LISTENER =
  (SID_LIST =
    (SID_DESC =
        (ORACLE_HOME=/opt/oracle/product/19c/dbhome_1)
        (SID_NAME =ALSSTGDR)
    )
  )
 ```
 * Update sys, system password or find system generated password for both primary and Standby
   * ALTER USER SYS IDENTIFIED BY ***;
   * ALTER USER SYSTEM IDENTIFIED BY ***;
 * Add standby redo logs on both primary and standby
   * ALTER DATABASE ADD STANDBY LOGFILE thread 1 group 21;
   * ALTER DATABASE ADD STANDBY LOGFILE thread 1 group 22;
   * ALTER DATABASE ADD STANDBY LOGFILE thread 1 group 23;
 * rman copy control file and datafiles from primary
    * rman target /
    * rman>startup nomount pfile='/opt/oracle/oradata/dbconfig/ALSSTGDR/pfileALSSTGDR.ora';
    * rman>exit
    * $rman (run rman withouth "target /" )
    * rman> connect target sys/welcome1@alsstgpri
    * rman> connect auxiliary sys/welcome1@alsstgstb
    * rman create Standby active database;
    ```
    run {
        allocate channel prmy1 type disk;
        allocate channel prmy2 type disk;
        allocate auxiliary channel stby1 type disk;
        allocate auxiliary channel stby2 type disk;
        duplicate target database for standby from active database;
    }
    ```

#### Setup Dataguard Broker
* Startup mount DB on spfile on Standby
  * create spfile
    * SQL> create spfile='/opt/oracle/oradata/dbconfig/ALSSTGDR/spfileALSSTGDR.ora' from pfile='/opt/oracle/oradata/dbconfig/ALSSTGDR/pfileALSSTGDR.ora';  
  * SQL>shutdown immediate
  * SQL>startup mount
* On both primary and standby side .
  * SQL> alter system set DG_BROKER_START=true scope=both;
* Create the Data Guard Broker configuration.
```
On primary side:
dgmgrl
DGMGRL> connect /
DGMGRL> create configuration ALSSTGDG as primary database is ALSSTG connect identifier is ALSSTGPRI;
DGMGRL> add database ALSSTGDR as connect identifier is ALSSTGSTB;
DGMGRL> show configuration;
DGMGRL> enable configuration;
DGMGRL> validate database ALSSTG;
DGMGRL> validate database ALSSTGDR ;
```
* Switchover steps
```
Connect to standby from dgmgrl
DGMGRL> connect /
DGMGRL> switchover to ALSSTGDR;
DGMGRL> show configuration;
DGMGRL> show database ALSSTGDR;
DGMGRL> show database ALSSTG ;
```
