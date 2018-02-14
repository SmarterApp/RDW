## Upgrade v1.1 <- v1.0

**Intended Audience**: this document provides detailed instructions for upgrading the [Reporting Data Warehouse (RDW)](../README.md) applications in an AWS environment from v1.0.x to v1.1. Operations and system administrators will find it useful.

It is assumed that the [Deployment Checklist](Deployment.AWS.md) was used for the installation of v1.0. Please refer to that documentation for general guidelines.

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

* [ ] Branch deployment and configuration repositories. All changes to these files will be made in the branch which can be quickly merged during the upgrade.
	```bash
	cd ~/git/../RDW_Deploy_Opus
	git checkout master; git pull
	git checkout -b v1_1 master
	git push -u origin v1_1
	
	cd ../RDW_Config_Opus
	git checkout master; git pull
	git checkout -b v1_1 master
	git push -u origin v1_1
	```
* [ ] Add a copy of this checklist to deployment and switch to that copy.
* [ ] Changes to deployment files. There are sample deployment files in the `deploy` folder in this repository; use those to copy or help guide edits.
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
			* Change image version to `1.1.0-RELEASE`
			* Reduce cpu/memory resources to `300m`/`750M`
			* Remove `volumeMounts` and `volumes` (copy over to new `reporting-service.yml`, next step)
			* Reduce replicas - 2 should be enough
		* [ ] Copy (new) `reporting-service.yml`
			* Add `volumeMounts` and `volumes` from old file (from previous step).
			* Set replicas to 4
		* [ ] Copy `aggregate-service.yml` and adjust if necessary (rare).
			* Set replicas to 1 (2 for HA only, single instance can saturate Redshift)
		* [ ] Copy (replace) `admin-service.yml` and adjust if necessary (rare). This completely replaces the old admin service.
			* Set replicas to 1 (2 for HA)
		* [ ] Edit `report-processor-service.yml` and set image version to `1.1.0-RELEASE`.
			* Leave replicas as 2
	* Commit changes
		```bash
		cd ~/git/../RDW_Deploy_Opus
		git commit -am "Changes for v1.1"
		git push 
		```
* [ ] Changes to configuration files. There are annotated configuration files in the `config` folder in this repository; use those to help guide edits.
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
			* Configure `spring.migrate_datasource` copy from migrate-reporting `warehouse_datasource`
			* Configure `spring.warehouse_datasource`, copy from migrate-reporting `warehouse_datasource`
			* Configure S3 archive, copy from import-service `archive`
			* Configure redshift role - TBD
			* Configure `spring.olap_datasource` - TBD
	* [ ] Reporting services.
    	* [ ] Admin service, copy `rdw-reporting-admin-service.yml` and edit
			* Configure `spring.rabbitmq`, copy from `rdw-admin-webapp.yml`
			* Configure `archive`, copy from `rdw-admin-webapp.yml`
			* Configure `spring.warehouse_datasource`, copy from `rdw-admin-webapp.yml`
			* Configure `spring.reporting_datasource`, copy from `rdw-reporting-webapp.yml`
		* [ ] Aggregate service, copy `rdw-reporting-aggregate-service.yml` and edit
			* Configure `spring.rabbitmq`, copy from admin service
			* Configure `app.archive`, copy from admin service
			* Configure `spring.olap_datasource` - TBD
		* [ ] Reporting service, copy `rdw-reporting-service.yml` and edit
			* Configure `app.iris.vendorId`, copy from `rdw-reporting-webapp.yml`
			* Configure `app.min-item-data-year`, copy from `rdw-reporting-webapp.yml`
			* Configure `artifacts`, copy from `rdw-reporting-webapp.yml`
			* Configure `spring.datasource`, copy from `rdw-reporting-webapp.yml`
			* Configure `cloud`, copy from `rdw-reporting-report-processor.yml` `app.archive.cloud` (note different property prefix)
		* [ ] Report processor, edit `rdw-reporting-report-processor.yml`
			* Configure `spring.rabbitmq`, copy from admin service
			* Move `app.datamart.datasource` to `spring.datasource` and replace `spring.datasource.url` with `spring.datasource.url-server`
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
                      url: http://report-processor
                    reporting-service:
                      url: http://reporting-service
              ```			  
			* Remove `app.admin-webapp-url`
			* Remove `app.reportservice`
			* Remove `app.wkhtmltopdf`
			* Remove `artifacts`
			* Remove `spring.rabbitmq`
			* Move `app.datamart.datasource` to `spring.datasource` and replace `spring.datasource.url` with `spring.datasource.url-server`
		* Delete `rdw-admin-webapp.yml`
		* Delete `rdw-reporting-admin-webapp.yml` (rare, will exist only if interim builds were deployed)
	* Commit changes
		```bash
		cd ~/git/../RDW_Config_Opus
		git commit -am "Changes for v1.1"
		git push 
		```		
* [ ] Update ops system, aka jump server. 
    * [ ] Install `psql` client, e.g. `sudo yum install postgresql`.
* [ ] Configure Redshift cluster. Redshift is expensive. Have to decide whether to share a single cluster between your environments.
	* Obviously this work will affect configuration files from previous steps.
	* Put Redshift in its own subnet (group?) in the production VPC and then VPC peer that subnet to allow staging, and jump servers to access it
	* Name like rdw-prod
	* Cluster should be 2 nodes, dc2.large
	* Cluster Parameter Group, e.g. rdw-prod-redshift10
		* Enable Short Query Acceleration, 5 seconds
		* Concurrency = 6
* [ ] Create Redshift role to grant access to S3.
	* [ ] Verify the existing policy that allows access to the S3 archive bucket. It should have a name like `AllowRdwOpusArchiveBucket`.
	* [ ] Create Redshift role with that policy. Use the console or CLI, and a consistent naming convention for your environment:
		```bash
		aws iam create-role --role-name redshiftRdwOpusArchiveAccess --description "Redshift access to rdw-opus-archive" --assume-role-policy-document file://RedshiftRoleTrustPolicy.json
		aws iam attach-role-policy --role-name redshiftRdwOpusArchiveAccess --policy-arn arn:aws:iam::478575410002:policy/AllowRdwOpusArchiveBucket
		```
	* [ ] Associate that role with the Redshift cluster. Use the console (Manage IAM roles) or CLI (TBD)
* [ ] Create Redshift database and users. To avoid excessive costs, a single cluster can be used to support multiple environments. This is done using a separate database for each environment. Using `psql`, create a database for the environment, the reporting schema and users
	```psql
	CREATE DATABASE opus;
	CREATE USER rdw-ingest PASSWORD 'AGoodPassword';
	CREATE USER rdw-reporting PASSWORD 'AnotherGoodPassword';
	-- are these necessary?
	-- GRANT ALL ON DATABASE opus TO rdw-ingest;
	-- GRANT ALL ON DATABASE opus TO rdw-reporting;
	\connect opus
	CREATE SCHEMA reporting;
	GRANT ALL ON SCHEMA reporting to rdw-ingest;
	-- can we just grant select for reporting?
	-- GRANT ALL ON SCHEMA reporting to rdw-reporting;
	GRANT SELECT ON ALL TABLES IN SCHEMA reporting TO rdw-reporting;
	ALTER USER rdw-ingest SET SEARCH_PATH TO reporting;
	ALTER USER rdw-reporting SET SEARCH_PATH TO reporting;
	```
* [ ] Create Roles / Permissions
	* All this is for the component `Reporting`
	* Create new roles and enable at appropriate levels: 
		* `Instructional Resource Admin` - State, District, GroupOfInstitutions, Institution
		* `Embargo Admin` - State, District
		* `Custom Aggregate Reporter` - State, District, GroupOfInstitutions, Institution
	* Create new permissions and map them to roles:
		* `CUSTOM_AGGREGATE_READ` - Custom Aggregate Reporter
		* `EMBARGO_READ` - Embargo Admin
		* `EMBARGO_WRITE` - Embargo Admin
		* `INSTRUCTIONAL_RESOURCE_WRITE` - Instructional Resource Admin
	* Chain admin roles to PII users


### Quiesce the System
Before upgrading the system it must be made idle.

* [ ] Verify 3rd party data feeds are suspended.
* [ ] Set up static landing page and route all site traffic to it.
	* reporting.sbac.org should redirect to static landing page
	* admin.sbac.org should go away
	* import.sbac.org can return an error during the upgrade
* [ ] Scale down all service deployments to have no pods. 
	```bash
	kubectl scale deployment import-deployment --replicas=0
	kubectl scale deployment package-processor-deployment --replicas=0
	kubectl scale deployment group-processor-deployment --replicas=0
	kubectl scale deployment exam-processor-deployment --replicas=0
	kubectl scale deployment migrate-reporting-deployment --replicas=0
	kubectl scale deployment task-server-deployment --replicas=0
	kubectl scale deployment admin-deployment --replicas=0
	kubectl scale deployment reporting-deployment --replicas=0
	kubectl scale deployment report-processor-deployment --replicas=0
	```
* We're on the clock. Play the Jeopardy theme in a loop ...	

### Backup
All cluster deployment and configuration is stored in version control, so nothing is necessary for that.

* [ ] Backup Aurora databases.


### Upgrade

* [ ] Delete current deployments as necessary. For many of the services, only the configuration has changed. However, some of the services have significant changes that require completely recreating them. NOTE: this is being done with the spec files **before** merging the branch.
	```bash
	kubectl delete -f admin-service.yml
	kubectl delete -f reporting-service.yml
	```
TODO - this will wipe the reporting service load balancer; that okay, did we customize it at all?
* [ ] Upgrade cluster (?)
	```bash
	kops upgrade cluster --name awsopus.sbac.org --state s3://kops-awsopus-sbac-org-state-store --yes
	# TODO - can we block rolling update and just slam out the upgrade? 
	```
* [ ] Increase cluster size
	* Not needed for production; might want to revisit auto-scaler configuration
* [ ] Apply schema changes.
	```bash
	# get latest version of the schema
	cd ~/git/../RDW_Schema
	git checkout master; git pull
	
	# test credentials and state of warehouse, then migrate it (this may take a while)
	gradle --Pdatabase_url="jdbc:mysql://rdw-aurora-warehouse-[aws-randomization]:3306/" \
   		-Pdatabase_user=user -Pdatabase_password=password infoWarehouse
	gradle --Pdatabase_url="jdbc:mysql://rdw-aurora-warehouse-[aws-randomization]:3306/" \
   		-Pdatabase_user=user -Pdatabase_password=password migrateWarehouse	
   # test credentials and state of reporting, then migrate it (this may take a while)
   gradle -Pdatabase_url="jdbc:mysql://rdw-aurora-reporting-[aws-randomization]:3306/" \
   		-Pdatabase_user=user -Pdatabase_password=password infoReporting
   gradle -Pdatabase_url="jdbc:mysql://rdw-aurora-reporting-[aws-randomization]:3306/" \
   		-Pdatabase_user=user -Pdatabase_password=password migrateReporting

	# create olap data mart
    gradle -Predshift_url=jdbc:redshift://rdw-[aws-randomization]:5439/opus \
    	-Predshift_user=user -Predshift_password=password \
    	-Pdatabase_url=jdbc:mysql://rdw-aurora-warehouse-[aws-randomization]:3306 \
    	-Pdatabase_user=user -Pdatabase_password=password \
    	migrateReporting_olap migrateMigrate_olap
	```
* [ ] Merge deployment and configuration branches. This can be done via command line or via the repository UI; if you use the repository UI, make sure to checkout and pull the latest `master`. Here are the command line actions:
	```bash
	cd ~/git/../RDW_Deploy_Opus
	git checkout v1_1; git pull
	git checkout master
	git merge v1_1
	git push origin master
	git push origin -d v1_1; git branch -d v1_1
	
	cd ~/git/../RDW_Config_Opus
	git checkout v1_1; git pull
	git checkout master
	git merge v1_1
	git push origin master
	git push origin -d v1_1; git branch -d v1_1
	```
* [ ] Redeploy services
	```bash
	cd ~/git/../RDW_Deploy_Opus
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
* [ ] Route 53
	* redo for reporting webapp (pretty sure it will get deleted)
	* remove for admin webapp
* [ ] Load new instructional resources	
	```bash
	TODO - sql command to run SQL; probably also need the actual SQL
	```
* [ ] Reload assessment packages. The tabulator output has been updated to include additional assessment data (especially for items). This upgrade supports updating assessment packages. So, submit the new tabulator output for all assessments. Note: the assessment labels are still wrong in the tabulator output so the `label_assessments` script must be rerun as well.
	```bash
	TODO - curl commands to POST CSVs
	```
	```sql
	TODO - mysql command for running script
	```
* [ ] Miscellaneous cleanup
	* [ ] Remove admin-webapp stuff from OpenAM
	* [ ] Remove DNS / route for admin-webapp (from Route 53, GoDaddy, wherever)

### Smoke Test
Smoke 'em if ya got 'em.		 

		 


