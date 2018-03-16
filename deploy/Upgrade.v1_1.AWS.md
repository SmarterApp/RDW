## Upgrade v1.1 <- v1.0

**Intended Audience**: this document provides detailed instructions for upgrading the [Reporting Data Warehouse (RDW)](../README.md) applications in an AWS environment from v1.0.x to v1.1. Operations and system administrators will find it useful.

It is assumed that the v1.0 [Deployment Checklist](https://github.com/SmarterApp/RDW/blob/1.0/deploy/Deployment.AWS.md) was used for the installation. Please refer to that documentation for general guidelines.

> Although these are generic instructions, having an example makes it easier to document and understand. Throughout this document we'll use `opus` as the sample environment name; this might correspond to `staging` or `production` in the real world. We'll use `sbac.org` as the domain/organization. Any other ids, usernames or other system-generated values are products of the author's imagination. The reader is strongly encouraged to use their own consistent naming conventions. **Avoid putting real environment-specific details _especially secrets and sensitive information_ in this document.**

### Overview

This is the first significant upgrade to RDW. It adds "Phase 2" functionality:
* Custom Aggregate Reports.
* Key / Distractor Analysis and Writing Trait Scores
* State/District-provided Instructional Links
* District/School groups (from ART).

It involves some significant architectural changes:
* Introduces Redshift as a data mart for aggregate reporting.
* Significant changes to the reporting services.
* Combine webapp applications (i.e. admin-webapp goes away)

Because there are schema changes, the data warehouse and reporting data mart must be migrated. This may take a couple hours depending on the amount of data involved. Combined with the non-trivial changes to the system configuration, this means the upgrade process may take 2-4 hours. It is important to alert the user base and any 3rd party data feeds of the extended downtime required for this upgrade.

The high level steps for the upgrade include:
* [Notification](#notification)
* [Prep Work](#prep-work)
* [Quiesce the System](#quiesce-the-system)
* [Backup](#backup)
* [Upgrade](#upgrade)
* [Smoke Test](#smoke-test)

### Notification
Well in advance of this work:

* [ ] Message user base about the upgrade and extended maintenance window.
* [ ] Alert 3rd party data feeds about the extended maintenance window.


### Prep Work
The goal of this step is to make changes to everything that doesn't directly affect current operations, leaving the absolute minimum work required during the actual maintenance window.

* These instructions expect you to have access to the files in the RDW project, so clone that locally if needed.
    ```bash
    cd ~/git
    git clone https://github.com/SmarterApp/RDW.git
    ```
* [ ] Branch deployment and configuration repositories. All changes to these files will be made in the branch which can be quickly merged during the upgrade.
    ```bash
    cd ~/git
    git clone https://github.com/SmarterApp/RDW_Deploy_Opus.git
    git clone https://github.com/SmarterApp/RDW_Config_Opus.git

    cd RDW_Deploy_Opus
    git checkout master; git pull
    git checkout -b v1_1 master
    git push -u origin v1_1
    
    cd ../RDW_Config_Opus
    git checkout master; git pull
    git checkout -b v1_1 master
    git push -u origin v1_1
    ```
* [ ] Add a copy of this checklist to deployment and switch to that copy.
* [ ] Changes to deployment files in `RDW_Deploy_Opus`. There are sample deployment files in the `deploy` folder in the `RDW` repository; use those to copy or help guide edits.
    * Common services. These helper services require no changes.
        * rabbit service
        * configuration service
        * wkhtmltopdf service
    * [ ] Ingest services. For most of these, only the image version needs to be changed. There is one new ingest service.
        * [ ] Change image version to `1.1.0-RELEASE` in the following files:
            * `import-service.yml`
            * `package-processor-service.yml`
            * `group-processor-service.yml`
            * `exam-processor-service.yml`
            * `migrate-reporting-service.yml`
            * `task-service.yml`
        * [ ] Copy `migrate-olap-service.yml` and adjust if necessary (rare).
    * [ ] Reporting services. Because of architecture changes in this release, there are more changes to make for these services.
        * [ ] Rename `reporting-service.yml` to `reporting-webapp.yml` and edit (compare to file in the `deploy` folder to help with this):
            * Change service name and app selector to `reporting-webapp`
            * Change deployment name to `reporting-webapp-deployment`
            * Change deployment spec app to `reporting-webapp`
            * Change image version to `1.1.1-RELEASE`
            * Reduce cpu/memory resources to `750m`/`1G`
            * Remove `volumeMounts` and `volumes` (copy over to new `reporting-service.yml`, next step)
        * [ ] Copy (new) `reporting-service.yml`
            * Add `volumeMounts` and `volumes` from old file (from previous step).
            * Set replicas to match reporting-webapp (perhaps -1 if there are lots)
        * [ ] Copy `aggregate-service.yml` and adjust if necessary (rare).
        * [ ] Copy (replace) `admin-service.yml` and adjust if necessary (rare). This completely replaces the old admin service.
        * [ ] Edit `report-processor-service.yml` and set image version to `1.1.1-RELEASE`.
    * Commit changes
        ```bash
        cd ~/git/RDW_Deploy_Opus
        git add *
        git commit -am "Changes for v1.1"
        git push 
        ```
* [ ] Changes to configuration files in `RDW_Config_Opus`. There are annotated configuration files in the `config` folder in the `RDW` repository; use those to help guide edits.
    * [ ] Common service properties. There are some properties that are used by many services. To avoid duplication and help with maintenance these should be set in the `application.yml` file. 
        * Copy or edit `application.yml` as appropriate, putting in values from the annotated sample, e.g.
            ```yaml
            app:
              school-year: 2018
              state:
                code: CA
                name: California
            
            tenant:
              transfer-access-enabled: true
            
            logging:
              level:
                # this outputs a couple INFO messages every 5 minutes, annoying
                org.springframework.cloud.config.client.ConfigServicePropertySourceLocator: warn
            ```
    * [ ] Ingest services. 
        * [ ] Import service, edit `rdw-ingest-import-service.yml`
            * replace `spring.datasource.url` with `spring.datasource.url-server`
        * [ ] Package processor, edit `rdw-ingest-package-processor.yml`
            * replace `spring.datasource.url` with `spring.datasource.url-server`
        * [ ] Group processor, edit `rdw-ingest-group-processor.yml`
            * replace `spring.datasource.url` with `spring.datasource.url-server`
        * [ ] Exam processor, edit `rdw-ingest-exam-processor.yml`
            * replace `spring.datasource.url` with `spring.datasource.url-server`
        * [ ] Migrate reporting, edit `rdw-ingest-migrate-reporting.yml`
            * replace `spring.reporting_datasource.url` with `spring.reporting_datasource.url-server`
            * replace `spring.warehouse_datasource.url` with `spring.warehouse_datasource.url-server`
        * [ ] Task service, edit `rdw-ingest-task-service.yml`
            * replace `spring.datasource.url` with `spring.datasource.url-server`
            * for `task.update-organizations` add `groups-of-districts-url` and `groups-of-schools-url`
        * [ ] Migrate olap, copy `rdw-ingest-migrate-olap.yml` and edit
            * Configure `spring.migrate_datasource` copy from `rdw-ingest-migrate-reporting.yml` `warehouse_datasource`
            * Configure `spring.warehouse_datasource`, copy from `rdw-ingest-migrate-reporting.yml` `warehouse_datasource`
            * Configure `migrate.olap-batch.run-cron`, make it after update organization task
            * Configure S3 archive, copy from `rdw-ingest-import-service.yml` `archive`
            * Configure `migrate.aws.redshift.role`, e.g. arn:aws:iam::[acct]:role/rdw-redshift-access
            * Configure `spring.olap_datasource`
                * `url-server`: rdw.[aws-randomization].redshift.amazonaws.com:5439
                * `url-db`: opus
                * `username`: rdwopusingest
                * `password`, encrypted using config server
    * [ ] Reporting services.
        * [ ] Admin service, copy `rdw-reporting-admin-service.yml` and edit
            * Configure `spring.rabbitmq`, copy from `rdw-admin-webapp.yml`
            * Configure `archive`, copy from `rdw-admin-webapp.yml`
            * Configure `spring.warehouse_datasource`, copy from `rdw-admin-webapp.yml` `spring.datasource`
            * replace `spring.warehouse_datasource.url` with `spring.warehouse_datasource.url-server`
            * Configure `spring.reporting_datasource`, copy from `rdw-reporting-webapp.yml` `app.datamart.datasource`
            * Replace `spring.reporting_datasource.url` with `spring.reporting_datasource.url-server`
        * [ ] Aggregate service, copy `rdw-reporting-aggregate-service.yml` and edit
            * Configure `spring.rabbitmq`, copy from `rdw-reporting-admin-service.yml`
            * Configure `app.archive`, copy from `rdw-reporting-admin-service.yml`
            * Configure `app.cache.repository.refresh-cron`, make it after migrate olap task
            * Configure concurrency for Redshift. In general the idea is to saturate the Redshift queue. With a single
            pod, do that by matching the pool size to the queue size and handling two requests simultaneously.
                * Configure `app.aggregate-reports.query-pool-size`, set to Redshift WLM concurrency, 12
                * Configure `spring.cloud.stream.bindings.AggregateRequest.consumer.concurrency`, set to 2
            * Configure `spring.olap_datasource`
                * `url-server`: rdw.[aws-randomization].redshift.amazonaws.com:5439
                * `url-db`: opus
                * `username`: rdwopusreporting
                * `password`, encrypted using config server
        * [ ] Reporting service, copy `rdw-reporting-service.yml` and edit
            * Configure `app.iris.vendorId`, copy from `rdw-reporting-webapp.yml`
            * Configure `app.min-item-data-year`, copy from `rdw-reporting-webapp.yml`
            * Configure `artifacts`, copy from `rdw-reporting-webapp.yml`
            * Configure `spring.datasource`, copy from `rdw-reporting-webapp.yml` `app.datamart.datasource`
            * Replace `spring.datasource.url` with `spring.datasource.url-server`
            * Configure `cloud`, copy from `rdw-reporting-report-processor.yml` `app.archive.cloud` (note different property prefix)
        * [ ] Report processor, edit `rdw-reporting-report-processor.yml`
            * Configure `spring.rabbitmq`, copy from `rdw-reporting-admin-service.yml`
            * Move `app.datamart.datasource` to `spring.datasource` 
            * Replace `spring.datasource.url` with `spring.datasource.url-server`
            * Remove `app.state.code`
            * Configure `task.remove-stale-reports` task, copy from annotated sample, something like:
                ```yaml
                task:
                  remove-stale-reports:
                    cron: 0 0 8 * * *
                    max-report-lifetime-days: 30
                    max-random-minutes: 20
                ```                
        * [ ] Reporting webapp, edit `rdw-reporting-webapp.yml`
            * Configure `zuul.routes`, copy from annotated sample, something like:
                ```yaml
                zuul:
                  routes:
                    admin-service:
                      url: http://admin-service
                    aggregate-service:
                      url: http://aggregate-service
                    report-processor:
                      url: http://report-processor-service
                    reporting-service:
                      url: http://reporting-service
              ```              
            * Remove `app.admin-webapp-url`
            * Remove `app.reportservice`
            * Remove `app.state.code`
            * Remove `app.wkhtmltopdf`
            * Remove `artifacts`
            * Remove `spring.rabbitmq`
            * Move `app.datamart.datasource` to `spring.datasource` 
            * Replace `spring.datasource.url` with `spring.datasource.url-server`
        * Delete `rdw-admin-webapp.yml`
        * Delete `rdw-reporting-admin-webapp.yml` (rare, will exist only if interim builds were deployed)
    * Commit changes
        ```bash
        cd ~/git/RDW_Config_Opus
        git add *
        git commit -am "Changes for v1.1"
        git push 
        ```        
* [ ] Configure Redshift cluster. Redshift is expensive. Have to decide whether to share a single cluster between your environments.
    * Obviously this work will affect configuration files from previous steps.
    * Cluster Parameter Group, e.g. rdw-opus-redshift10
        * Enable Short Query Acceleration, 5 seconds
        * Concurrency = 12
    * Create Cluster Subnet Group in opus VPC, rdw-opus is a good name, add multiple zones
    * Create Security Group
        * rdw-redshift
        * opus VPC
        * redshift traffic from nodes (use security group or CIDRs) and jump server 
    * Name like rdw-opus
    * Cluster should be 2 nodes, dc2.large
    * Put Redshift in opus VPC using cluster subnet group and other stuff we just created
* [ ] Create Redshift role to grant access to S3.
    * [ ] Verify the existing policy that allows access to the S3 archive bucket. It should have a name like `AllowRdwOpusArchiveBucket`.
    * [ ] Create Redshift role with that policy. Use the console or CLI, and a consistent naming convention for your environment:
        ```bash
        aws iam create-role --role-name redshiftRdwOpusArchiveAccess --description "Redshift access to rdw-opus-archive" --assume-role-policy-document file://RedshiftRoleTrustPolicy.json
        aws iam attach-role-policy --role-name redshiftRdwOpusArchiveAccess --policy-arn arn:aws:iam::478575410002:policy/AllowRdwOpusArchiveBucket
        ```
    * [ ] Associate that role with the Redshift cluster. Use the console (Manage IAM roles) or CLI (TBD)
* [ ] Update ops system, aka jump server. 
    * [ ] Install `psql`, e.g. `sudo yum install postgresql`. Install just the client, not the server! After installing
    check the connection to Redshift.
        ```bash
        psql --host=rdw-opus.[aws-randomization] --port=5439 --username=root --password --dbname=dev
        ```
    * [ ] (Optional) Configure psql to automatically connect. One way to do this ...
        * Create `~/.pg_service.conf` with something like this: 
            ```ini
            [rdw]
            host=rdw-opus.[aws-randomization]
            port=5439
            dbname=dev
            user=root
            password=password
            ```
        NOTE: there is a password in there so do something like `chmod 0600 ~/.pg_service.conf` to protect it.
        * Add to `.bashrc` (and source it)
            ```bash
            export PGSERVICE=rdw 
            ```
* [ ] Create Redshift database and users. To avoid excessive costs, a single cluster can be used to support multiple environments. This is done using a separate database for each environment. Using `psql`, create a database for the environment, the reporting schema and users
    ```sql
    CREATE USER rdwopusingest PASSWORD 'AGoodPassword';
    CREATE USER rdwopusreporting PASSWORD 'AnotherGoodPassword';
    CREATE DATABASE opus;
    ALTER DATABASE opus OWNER TO rdwopusingest;
    \connect opus
    CREATE SCHEMA reporting;
    GRANT ALL ON SCHEMA reporting to rdwopusingest;
    GRANT ALL ON ALL TABLES IN SCHEMA reporting to rdwopusingest;
    GRANT ALL ON SCHEMA reporting to rdwopusreporting;
    GRANT ALL ON ALL TABLES IN SCHEMA reporting to rdwopusreporting;
    ALTER USER rdwopusingest SET SEARCH_PATH TO reporting;
    ALTER USER rdwopusreporting SET SEARCH_PATH TO reporting;
    ```
* [ ] Create reporting olap schemas. There is the schema in Redshift and there is a migrate olap schema in the same
Aurora instance as the warehouse schema. Use RDW_Schema project to create the schema. NOTE: this would normally use the 
`master` branch but the `develop` branch had not yet been merged when these instructions were written; please adjust 
to use whichever branch corresponds to the v1.1 release.
    ```bash
    cd RDW_Schema
    # see NOTE above
    git checkout develop; git pull
    # create schemas
    ./gradlew -Pdatabase_url="jdbc:mysql://rdw-opus-warehouse-cluster.[aws-randomization]:3306/" \
      -Pdatabase_user=root -Pdatabase_password=password \
      -Predshift_url=jdbc:redshift://rdw-opus.[aws-randomization]:5439/opus \
      -Predshift_user=root -Predshift_password=password \
      migrateMigrate_olap migrateReporting_olap
    ``` 
    * [ ] In Aurora, grant access to the migrate olap schema to the existing ingest database user, and allow both users 
    to write to S3 (ingest needs it for migrate, reporting needs it for report generation)..
        ```bash
        mysql -h rdw-opus-warehouse-cluster.[aws-randomization] -u root -p
        mysql> grant all privileges on migrate_olap.* to 'rdw-ingest'@'%';
        mysql> grant SELECT INTO S3 ON *.* to 'rdw-ingest'@'%';
        
        mysql -h rdw-opus-reporting-cluster.[aws-randomization] -u root -p
        mysql> grant SELECT INTO S3 ON *.* to 'rdw-ingest'@'%';
        mysql> grant SELECT INTO S3 ON *.* to 'rdw-reporting'@'%';
        ```
    * [ ] Verify Redshift user search path by using their credentials to connect and list tables (or something similar).
        ```bash
        psql --host=rdw-opus.[aws-randomization] --port=5439 --username=rdwopusreporting --password --dbname=opus
        psql --host=rdw-opus.[aws-randomization] --port=5439 --username=rdwopusingest --password --dbname=opus
  
        stage=> \dt
                              List of relations
          schema   |               name               | type  | owner 
        -----------+----------------------------------+-------+-------
         reporting | administration_condition         | table | root
         reporting | asmt                             | table | root
        ...
        ```
* [ ] Test connectivity between S3, Aurora and Redshift. The data migration uses S3 to stage data between the warehouse
and the data marts. This is a good time to verify that the required connectivity and permissions are in place for that.
    * Export test file from Aurora. Connect to the warehouse using the ingest user and export a table to a test file.
        ```bash
        mysql -h rdw-opus-warehouse-cluster.[aws-randomization] --user=rdw-ingest --password=password warehouse
        mysql> SELECT id, code FROM completeness INTO OUTFILE S3 's3://rdw-opus-archive/completeness' FIELDS TERMINATED BY ',' OPTIONALLY ENCLOSED BY '"' LINES TERMINATED BY '\n';
        Query OK, 2 rows affected (0.13 sec)
        ```
    * Import test file to Redshift. Connect to redshift using the ingest user and import the test file into the table.
        ```bash
        psql --host=rdw-opus.[aws-randomization] --port=5439 --username=rdwopusingest --password --dbname=opus
        opus=> COPY reporting.staging_completeness (id, code) FROM 's3://rdw-opus-archive/completeness.part_00000' CREDENTIALS 'aws_iam_role=arn:aws:iam::[aws-randomization]' FORMAT AS CSV DELIMITER ',' COMPUPDATE OFF;
        INFO:  Load into table 'staging_completeness' completed, 2 record(s) loaded successfully.
        opus=> select * from staging_completeness;
         id |   code   
        ----+----------
          1 | Partial
          2 | Complete
        (2 rows)
        opus=> delete from staging_completeness;
        DELETE 2
        opus=> \q
        ```
* [ ] Create Roles / Permissions
    * All this is for the component `Reporting`
    * [ ] Create new roles and enable at appropriate levels: 
        * `Instructional Resource Admin` - State, District, GroupOfInstitutions, Institution
        * `Embargo Admin` - State, District
        * `Custom Aggregate Reporter` - State, District, GroupOfInstitutions, Institution
    * [ ] Create new permissions and map them to roles:
        * `CUSTOM_AGGREGATE_READ` - Custom Aggregate Reporter
        * `EMBARGO_READ` - Embargo Admin
        * `EMBARGO_WRITE` - Embargo Admin
        * `INSTRUCTIONAL_RESOURCE_WRITE` - Instructional Resource Admin
    * (?) Chain admin roles to PII users
* [ ] (Optional) "Before" Smoke Test. You may want to go through some of the steps of the smoke test before doing the
upgrade, just to make sure any problems are new. This may require temporarily providing access for your QA volunteers.


### Quiesce the System
Before upgrading the system it must be made idle.

* [ ] Verify 3rd party data feeds are suspended.
* [ ] Set up static landing page and route all site traffic to it.
    * reporting.sbac.org should redirect to static landing page
    * admin.sbac.org should go away
    * import.sbac.org can return an error during the upgrade
* [ ] Disable any ops monitoring on RDW services.
* We're on the clock. Play the Jeopardy theme in a loop ...    
* [ ] Scale down all service deployments to have no pods. The first three deployments listed allow/cause data changes in
the system. It is suggested that scale them down, wait 5 minutes to all migration to complete, then scale down the rest. 
    ```bash
    # these allow/cause data changes
    kubectl scale deployment task-server-deployment --replicas=0
    kubectl scale deployment admin-deployment --replicas=0
    kubectl scale deployment import-deployment --replicas=0
    # pause here for 2-3 minutes to allow processing and migration to complete 
    kubectl scale deployment package-processor-deployment --replicas=0
    kubectl scale deployment group-processor-deployment --replicas=0
    kubectl scale deployment exam-processor-deployment --replicas=0
    kubectl scale deployment migrate-reporting-deployment --replicas=0
    kubectl scale deployment reporting-deployment --replicas=0
    kubectl scale deployment report-processor-deployment --replicas=0
    ```

### Backup
All cluster deployment and configuration is stored in version control, so nothing is necessary for that.

* [ ] Backup Aurora databases.


### Upgrade

* [ ] Get the latest deploy repo
    ```bash
    cd ~/git/RDW_Deploy_Opus
    git checkout master; git pull
    ```
* [ ] Delete current deployments as necessary. For many of the services, only the configuration has changed. However, some of the services have significant changes that require completely recreating them. NOTE: this is being done with the spec files **before** merging the branch.
    ```bash
    kubectl delete -f admin-service.yml
    kubectl delete -f reporting-service.yml
    ```
* [ ] Upgrade cluster. If the version of the cluster is old (< 1.7 at the time of this writing), consider upgrading it.
    ```bash
    kops upgrade cluster --name awsopus.sbac.org --state s3://kops-awsopus-sbac-org-state-store --yes
    ```
* [ ] Increase cluster size. If the cluster is just barely big enough now, you'll need to add a node.
    * Not needed for production; might want to revisit auto-scaler configuration
* [ ] Apply schema changes. If the warehouse and reporting databases are separate it will be more efficient to run the
two migration tasks in parallel. Use two terminal sessions (or `screen`) and run them at the same time.
    ```bash
    # get latest version of the schema
    cd ~/git/RDW_Schema
    git checkout master; git pull
    
    # test credentials and state of databases
    ./gradlew -Pdatabase_url="jdbc:mysql://rdw-aurora-warehouse-[aws-randomization]:3306/" -Pdatabase_user=user -Pdatabase_password=password infoWarehouse
    ./gradlew -Pdatabase_url="jdbc:mysql://rdw-aurora-reporting-[aws-randomization]:3306/" -Pdatabase_user=user -Pdatabase_password=password infoReporting
    
    # migrate warehouse (this may take a while)
    ./gradlew -Pdatabase_url="jdbc:mysql://rdw-aurora-warehouse-[aws-randomization]:3306/" -Pdatabase_user=user -Pdatabase_password=password migrateWarehouse    
    # migrate reporting (this may take a while)
    ./gradlew -Pdatabase_url="jdbc:mysql://rdw-aurora-reporting-[aws-randomization]:3306/" -Pdatabase_user=user -Pdatabase_password=password migrateReporting
    ```
* [ ] Merge deployment and configuration branches. This can be done via command line or via the repository UI (if you use the repository UI, make sure to checkout and pull the latest `master`). Here are the command line actions:
    ```bash
    cd ~/git/RDW_Deploy_Opus
    git checkout v1_1; git pull
    git checkout master
    git merge v1_1
    git push origin master
    git push origin --delete v1_1; git branch -d v1_1
    
    cd ~/git/RDW_Config_Opus
    git checkout v1_1; git pull
    git checkout master
    git merge v1_1
    git push origin master
    git push origin --delete v1_1; git branch -d v1_1
    ```
* [ ] Redeploy services. 
    ```bash
    cd ~/git/RDW_Deploy_Opus
    # ingest services
    kubectl apply -f exam-processor-service.yml
    kubectl apply -f group-processor-service.yml
    kubectl apply -f package-processor-service.yml
    kubectl apply -f import-service.yml
    kubectl apply -f migrate-reporting-service.yml
    kubectl apply -f migrate-olap-service.yml
    kubectl apply -f task-service.yml
    # reporting services
    kubectl apply -f admin-service.yml
    kubectl apply -f aggregate-service.yml
    kubectl apply -f reporting-service.yml
    kubectl apply -f report-processor-service.yml
    kubectl apply -f reporting-webapp.yml
   ```
Check the logs of the services. Of particular interest are the migrate-olap-service which will immediately start migrating from the warehouse to the new aggregate reporting data mart, and the reporting-webapp which needs to talk to lots of other services.    
* [ ] Route 53
    * verify for import
    * redo for reporting webapp
    * remove for admin webapp
* [ ] Load data
    * [ ] Reload assessment packages. The tabulator output has been updated to include additional assessment data (especially for items). This upgrade supports updating assessment packages. So, submit the new tabulator output for all assessments. Note: the assessment labels are still wrong in the tabulator output so the `label_assessments` script must be rerun as well.
        ```bash
        # fetch an access token for loading data
        export ACCESS_TOKEN=`curl -s -X POST --data 'grant_type=password&username=rdw-ingest-opus@sbac.org&password=password&client_id=rdw&client_secret=password' 'https://sso.sbac.org/auth/oauth2/access_token?realm=/sbac' | jq -r '.access_token'`

        # load updated packages    
        curl -X POST --header "Authorization: Bearer ${ACCESS_TOKEN}" -F file=@2016v2.csv https://import.sbac.org/packages/imports
        curl -X POST --header "Authorization: Bearer ${ACCESS_TOKEN}" -F file=@2017v2.csv https://import.sbac.org/packages/imports
        curl -X POST --header "Authorization: Bearer ${ACCESS_TOKEN}" -F file=@2018v2.csv https://import.sbac.org/packages/imports
  
        # (Optional) check package processor logs
        kubectl logs -f package-processor-[k8s-randomization]
        ```
    * [ ] Run SQL for updating assessment labels
        ```bash
        # adjust labels and trigger migration
        mysql -h rdw-opus-warehouse.cimuvo5urx1e.us-west-2.rds.amazonaws.com -uroot --ppassword < label_assessments.sql
        ```
    * [ ] Run SQL for loading new instructional resources
        ```bash
        mysql -h rdw-opus-reporting.cimuvo5urx1e.us-west-2.rds.amazonaws.com -uroot -ppassword < load_instructional_resources.sql
        ```
* [ ] Miscellaneous cleanup
    * [ ] Restore traffic to site (from static web page)
    * [ ] Notify 3rd party to restore data feeds
    * [ ] Reenable any ops monitoring (remember to clean up any admin-webapp monitoring)
    * [ ] Remove admin-webapp stuff from OpenAM
    * [ ] Remove DNS / route for admin-webapp (from Route 53, GoDaddy, wherever)
    * [ ] Deploy new User Guide and Interpretive Guide

### Smoke Test
Smoke 'em if ya got 'em.         

NOTE: the system will automatically migrate data to the reporting and aggregate data marts. However, this process may
take a while and the system behavior will be a bit wonky during that time:
    * Migrate all ICA and Summative results to aggregate data mart. Less than an hour.
    * Migrate all students, now with inferred school, to the reporting data mart. Many (4-5?) hours. This blocks all 
    other migrations: assessment updates, student group uploads, new test results.
    * Migrate assessment changes to the reporting data mart. Seconds, but *after* students are migrated.

### Performance and Scalability
Changes to the reporting services have affected deployment specifications. The main reporting webapp with 750m CPU and
1G memory will support about 2000 concurrent users. The reporting service supports the individual test result queries 
and should be able to support about the same, 2000 concurrent users per pod with 500m CPU and 750M memory. The other
services should stay the same as the previous version.

NOTE: When allocating pod memory please consider the ratio of memory/processor for the nodes. It is silly to restrict
memory if it is just going to go to waste because of CPU allocation. So, if the nodes have a 4/1 memory to processor
ratio then the allocation should be similarly scaled: using the webapp as an example, it requires 750m processor so
giving it 4*750m = 3G memory would be okay. That said, none of these services will do much with >2G, so cap it.
 
Redshift is the backing OLAP data mart for the aggregate reports. Redshift deals with lots of data and is not 
particularly good at high concurrency. The RDW architecture attempts to mitigate this but know that Redshift itself
is the bottleneck when lots of requests are being made. Adding more aggregate-report pods will actually degrade 
performance. And a larger Redshift instance will provide only incremental improvements to overall performance of the
system. Given the cost of the larger instance, it is not advised to increase the size. The aggregate-report pod pool 
size should be coordinated with the Redshift queue size (WLM concurrency). Know that a report request typically needs 
less than 10 queries so, if the queue size is greater than that, the consumer concurrency should be increased so the 
Redshift queue is saturated. A couple examples:
    * Typical configuration.
        * Redshift queue size = 12
        * Aggregate-report service pool size = 12
        * Aggregate-report consumer concurrency = 2
    * Small configuration.
        * Redshift queue size = 6
        * Aggregate-report service pool size = 6
        * Aggregate-report consumer concurrency = 1
    * Large configuration.
        * Redshift queue size = 30
        * Aggregate-report service pool size = 30
        * Aggregate-report consumer concurrency = 3
    * Large configuration, alternate (not extensively tested)
        * Redshift queue size = 30
        * Aggregate-report service pods = 2
        * Aggregate-report service pool size = 15
        * Aggregate-report consumer concurrency = 2