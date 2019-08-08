## MultiTenancy

**Intended Audience**: This document contains information on how multi-tenancy works in the [Reporting Data Warehouse](../README.md) (RDW). Operations and system administration will find it useful.

### Table of Contents

* [Overview](#overview)
* [Creating Sandbox Data Sets](#creating-sandbox-data-sets)
* [Manual Tenant Creation](#manual-tenant-creation)
* [Manual Sandbox Creation](#manual-sandbox-creation)
* [Manual Configuration Change](#manual-configuration-change)

### Overview

[Architecture](Architecture.md#multi-tenancy)

A single instance of RDW can host multiple tenants, each with partitioned data.

#### Resources

Each additional tenant impacts the resources of RDW. If there are many 
tenants in a single instance, CPU and memory allocations for the services 
will have to be increased. The exact changes depend on the amount of 
data for each tenant, the number of concurrent users, etc.

Monitor the service logs for memory errors and the orchestration environment 
for container OOM restarts.

#### Sandbox Permissions

When users enter a sandbox they are granted a role based on the sandbox role they select:
* Teacher - `SandboxTeacher`
* School Admin - `SandboxSchoolAdmin`
* District Admin - `SandboxDistrictAdmin`

The permissions for these roles may be configured as desired in the permissions service however the recommendation is:
* SandboxTeacher - GROUP_PII_READ, GROUP_READ
* SandboxSchoolAdmin - INDIVIDUAL_PII_READ, GROUP_PII_READ, GROUP_READ, CUSTOM_AGGREGATE_READ
* SandboxDistrictAdmin - INDIVIDUAL_PII_READ, GROUP_PII_READ, CUSTOM_AGGREGATE_READ, GROUP_READ, GROUP_WRITE, EMBARGO_READ, EMBARGO_WRITE, INSTRUCTIONAL_RESOURCE_WRITE

NOTE: the names of the roles are not configurable.

### Creating Sandbox Data Sets

When creating a Sandbox, the system allows the administrator to select 
from available data sets. A data set is a collection of database table 
contents that are loaded into the warehouse (and migrated to the 
reporting databases). 
 
Unfortunately, at this time, creating the data sets is a manual process.

The goal is to load the warehouse with all the required data, without 
allowing the system to migrate the data. Then dump that data and stage 
it in the data sets folder in S3. Probably the easiest environment to
do this is a local development system.

For this example, we are creating a data set for Smarter Balanced 
assessments for ELA and Math using the demo institutions and a state
code of TS.

* Data generation
    * collect inputs for data generation
        * assessment package definitions
        * institution hierarchy
    * use data generator to create TRTs
    * spot check the TRTs
* Configure RDW services. This assumes you are pointing to a configuration with tenant-TS defined.
    * Reset the schema using RDW_Schema. Adjust school_year for generated data.
    ```
    RDW_Schema$ gradle -Pschema_prefix=ts_ cleanAll migrateAll
    RDW_Schema$ echo "DELETE FROM ts_warehouse.school_year WHERE year IN (2015, 2016)" | mysql -u root ts_warehouse
    RDW_Schema$ echo "INSERT INTO state_embargo (school_year, individual, aggregate, updated_by) VALUES (2019, 0, 0, 'dwtest@example.com')" | mysql -u root ts_warehouse
    ```
    * Run ingest services: modify RDW_Ingest docker-compose file, commenting out migrate-reporting and task-service:
    ```
    RDW_Ingest$ gradle clean buildImage
    RDW_Ingest$ docker-compose up -d
    ```
* Load data (the usual data load process using curl or Postman and data generator submit scripts). 
Be sure to use credentials for the `TS` tenant.
    * subject definitions
    * assessment packages
    * institutions
    * TRTs
```
export ACCESS_TOKEN="sbac;dwtest@example.com;|TS|ASMTDATALOAD|STATE|SBAC||||TS||||||||||"
curl -X POST -s --header "Authorization:Bearer ${ACCESS_TOKEN}" -F file=@ELA_subject.xml http://localhost:8080/subjects/imports
curl -X POST -s --header "Authorization:Bearer ${ACCESS_TOKEN}" -F file=@Math_subject.xml http://localhost:8080/subjects/imports
curl -X POST -s --header "Authorization:Bearer ${ACCESS_TOKEN}" -F file=@AccessibilityConfig.2019.xml http://localhost:8080/accommodations/imports
curl -X POST -s --header "Authorization:Bearer ${ACCESS_TOKEN}" -F file=@2017v3.csv http://localhost:8080/packages/imports
... repeat for all assessment packages
curl -X POST -s --header "Authorization:Bearer ${ACCESS_TOKEN}" -F file=@organizations.json http://localhost:8080/organizations/imports
# use submit_helper to submit TRTs (tweak ACCESS_TOKEN in script)
find ./out/*/*/* -type d | xargs -I FOLDER -P 3 ./scripts/submit_helper.sh FOLDER
```    
* Create groups. This is a tricky. We want a group per school per grade per subject.
We can use the session id from the data generator to group students, and
we'll base the groups on summative assessments for the current school year.
We'll restrict to medium sized groups.
All of this is dependent on the generated data so adjust the query populating
the temporary table as necessary ...
```
INSERT INTO import (status, content, contentType, digest, creator) VALUES (0, 5, 'text/plain', left(uuid(), 8), 'dwtest@example.com');
SELECT LAST_INSERT_ID() into @importid;

CREATE TEMPORARY TABLE school_grade_session
(tmpid INTEGER NOT NULL AUTO_INCREMENT, PRIMARY KEY (tmpid), INDEX(tmpid), UNIQUE (school_id, grade_id, subject_id))
IGNORE SELECT e.school_id, e.grade_id, a.subject_id, LEFT(e.session_id, 3) AS session, count(*) AS cnt
 FROM exam e JOIN asmt a ON e.asmt_id = a.id
 WHERE e.school_year=2019 AND a.type_id=3
 GROUP BY e.school_id, e.grade_id, a.subject_id, LEFT(e.session_id,3)
 HAVING cnt > 20 and cnt < 50;

INSERT INTO student_group (id, name, school_id, school_year, subject_id, active, creator, import_id, update_import_id)
SELECT tmpid, CONCAT_WS(' ', sc.name, 'Grade', TRIM(LEADING '0' FROM g.code), su.code) as name, school_id, 2019 as school_year, subject_id, 1 as active, 'dwtest@example.com' as creator, @importid as import_id, @importid as update_import_id
FROM school_grade_session sgs
JOIN school sc ON sgs.school_id = sc.id
JOIN grade g ON sgs.grade_id = g.id
JOIN subject su ON sgs.subject_id = su.id;

INSERT IGNORE INTO student_group_membership (student_group_id, student_id)
SELECT tmpid as student_group_id, e.student_id
FROM school_grade_session sgs
JOIN exam e ON sgs.school_id = e.school_id AND sgs.grade_id = e.grade_id AND e.school_year = 2019 AND LEFT(e.session_id, 3) = sgs.session
JOIN asmt a ON e.asmt_id = a.id AND a.type_id = 3 AND sgs.subject_id = a.subject_id;

UPDATE import SET status=1 WHERE id=@importid;

DROP TABLE school_grade_session;
```
* Dump data, cleanup and create manifest
```
mkdir -p /tmp/sbac-dataset
mysqldump -u root --tab=/tmp/sbac-dataset ts_warehouse
cd /tmp/sbac-dataset
rm schema_version.*
rm *.sql
rm audit_*.txt
find . -size 0 -delete
find *.txt > manifest.txt
```
* (Optional) Create a file describing the dataset, e.g. README
* Upload the files to S3, e.g. `s3://rdw-qa-archive/sandbox-datasets/sbac-dataset/warehouse`
* Annotate the dataset in the admin service configuration file, e.g.
```
# in rdw-reporting-admin-service.yml
sandbox-properties:
  sandboxDatasets:
    - label: SBAC Dataset
      id: sbac-dataset
```

### Manual Tenant Creation

The admin UI allows a user to create a tenant. It handles all the magic under the covers.
However, there may be situations where manually creating a tenant is necessary.

1. Know the tenant id/key. For this example we'll use `TS` (Test State).
1. Provision database resources. You'll need to do this in Aurora and Redshift. If there are multiple data servers these steps will be slightly different.
```sql
-- Aurora (mysql client)
CREATE DATABASE warehouse_ts DEFAULT CHARACTER SET 'utf8' COLLATE 'utf8_unicode_ci';
CREATE DATABASE reporting_ts DEFAULT CHARACTER SET 'utf8' COLLATE 'utf8_unicode_ci';
CREATE DATABASE migrate_olap_ts DEFAULT CHARACTER SET 'utf8' COLLATE 'utf8_unicode_ci';
CREATE USER 'rdw_ts_ingest'@'%' IDENTIFIED BY 'password';
CREATE USER 'rdw_ts_reporting'@'%' IDENTIFIED BY 'password';
GRANT ALL PRIVILEGES ON warehouse_ts.* TO 'rdw_ts_ingest'@'%';
GRANT ALL PRIVILEGES ON migrate_olap_ts.* TO 'rdw_ts_ingest'@'%';
GRANT ALL PRIVILEGES ON reporting_ts.* to 'rdw_ts_ingest'@'%';
GRANT ALL PRIVILEGES ON reporting_ts.* to 'rdw_ts_reporting'@'%';
GRANT SELECT ON mysql.proc TO 'rdw_ts_ingest'@'%';

-- Redshift (postgresql client)
\connect opus
CREATE SCHEMA reporting_ts;
CREATE USER rdw_ts_ingest PASSWORD 'password';
CREATE USER rdw_ts_reporting PASSWORD 'password';
GRANT ALL ON SCHEMA reporting_ts to rdw_ts_ingest;
GRANT ALL ON SCHEMA reporting_ts to rdw_ts_reporting;
ALTER USER rdw_ts_ingest SET SEARCH_PATH TO reporting_ts;
ALTER USER rdw_ts_reporting SET SEARCH_PATH TO reporting_ts;
```
1. Create the RDW schemas. The RDW_Schema project may be used for this.
```bash
git clone https://github.com/SmarterApp/RDW_Schema.git
cd RDW_Schema
git checkout master; git pull

# note the schema prefix; it can be anything but matching tenant id makes sense
gradle \
  -Pschema_suffix=_ts \
  -Pdatabase_url="jdbc:mysql://rdw-aurora-opus.cugsexobhx8t.us-west-2.rds.amazonaws.com:3306/" \
  -Pdatabase_user=rdw_ts_ingest -Pdatabase_password=password \
  -Predshift_url=jdbc:redshift://rdw-opus.cibkulpjrgtr.us-west-2.redshift.amazonaws.com:5439/opus \
  -Predshift_user=rdw_ts_ingest -Predshift_password=password \
  migrateWarehouse migrateReporting migrateMigrate_olap migrateReporting_olap
```
1. Finish up database resource permissions.
```sql
-- Aurora (mysql client)
GRANT LOAD FROM S3 ON *.* TO 'rdw_ts_ingest'@'%';
GRANT SELECT INTO S3 ON *.* TO 'rdw_ts_ingest'@'%';
GRANT SELECT INTO S3 ON *.* to 'rdw_ts_reporting'@'%';

-- Redshift (postgresql client)
\connect opus
GRANT ALL ON ALL TABLES IN SCHEMA reporting_ts to rdw_ts_ingest;
GRANT ALL ON ALL TABLES IN SCHEMA reporting_ts to rdw_ts_reporting;
```
1. Create configuration profile.
```bash
git clone https://gitlab.com/fairwaytech/rdw_config_opus.git
cd rdw_config_opus
git checkout master

mkdir tenant-TS
touch tenant-TS/application.yml
# edit application.yml; best bet is to copy an existing tenant's file and modify
git commit -am "Add TS tenant"
git push
```
1. Poke configuration service
```bash
# wait 10-15 seconds before and 20-30 seconds after this command
kubectl exec -it configuration-deployment-... -- curl -d 'path=tenant-TS' http://localhost:8888/monitor
```
1. Poke migrate services to migrate for the new tenant. The `migrate-reporting` service should do this right away but `migrate-olap` will have to be triggered. 
First verify the service has recognized the new tenant:
```bash
kubectl exec -it migrate-olap-... -- curl -X GET http://localhost:8008/migrate |sort
Migrate: running
OT: enabled; last migrate: COMPLETED to 2019-07-25T18:25:03.935443
TS: enabled
```
If that looks good, trigger the migrate for `TS`:
```bash
kubectl exec -it migrate-olap-... -- curl -X POST http://localhost:8008/migrate?tenantId=TS
```
1. After migrate has completed, trigger another configuration refresh (the alternative is to bounce all the services that aggressively cache information):
```
kubectl exec -it configuration-deployment-... -- curl -d 'path=*' http://localhost:8888/monitor
```

### Manual Sandbox Creation

The admin UI allows a user to create a sandbox. It handles all the magic under the covers.
However, there may be situations where manually creating a sandbox is necessary.
The steps are very similar to creating a tenant with some important differences:
* The sandbox id should follow the convention of the parent tenant id concatenated with "_Snnn" where nnn goes from 001 to 999. For example, a new sandbox for the TS tenant might have id TS_S012.
* For database resources, use the convention of a suffix qualifier, for example the warehouse database would be like `warehouse_ts_s001`
* For database users, prefer the naming convention of the lowercased, collapsed sandbox ID, for example `tss001`
* A sandbox configuration must indicate that it is a sandbox and include the dataset name.
```
tenantProperties:
  tenants:
    TS_S001:
      id: TS_S001
      key: TS_S001
      name: Test Sandbox 001
      sandboxDataset: "test-dataset"
      sandbox: true      
```
* The dataset must be loaded. The dataset should be in S3. Do this after the database setup steps and before creating the sandbox configuration profile.
```sql
-- Aurora (mysql client)
USE warehouse_ts_s001;
SET FOREIGN_KEY_CHECKS=0;

-- A dataset has a manifest.txt file; for each entry in that file do something like:
TRUNCATE table warehouse_ts_s001.accommodation;
LOAD DATA FROM S3 's3://rdw-opus-archive/sandbox-datasets/demo-dataset/warehouse/accommodation.txt' INTO TABLE warehouse_ts_s001.accommodation;

SET FOREIGN_KEY_CHECKS=1;
```
A hint for generating all the `LOAD DATA` commands (not quite right since the table name has `.txt` appended):
```bash
cat manifest.txt | xargs -n 1 -I FILE echo "LOAD DATA FROM S3 's3://rdw-opus-archive/sandbox-datasets/demo-dataset/warehouse/FILE' INTO TABLE warehouse_ts_s001.FILE"
```


### Manual Configuration Change

As part of the multi-tenant support, all the RDW services now respond dynamically to configuration changes. If a manual change is made to configuration, trigger the system to see the change:
```bash
kubectl exec -it configuration-deployment-<k8s-id> -- curl -d 'path=*' http://localhost:8888/monitor
```
It takes 20-30 seconds for the change to be loaded and propagated to the services.

