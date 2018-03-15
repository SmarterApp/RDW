# Runbook

**Intended Audience**: the runbook describes behavior and configuration options of the various applications in the [Reporting Data Warehouse](../README.md) (RDW). Operations, system administration, developers, and tier 3 support may find it useful.

### Table of Contents

1. [Common Conventions](#common)
1. [Import Service](#import-service)
1. [Package Processor](#package-processor)
1. [Exam Processor](#exam-processor)
1. [Group Processor](#group-processor)
1. [Migrate Reporting](#migrate-reporting)
1. [Migrate OLAP](#migrate-olap)
1. [Task Service](#task-service)
1. [Reporting Web App](#reporting-webapp)
1. [Reporting Service](#reporting-service)
1. [Aggregate Service](#aggregate-service)
1. [Report Processor](#report-processor)
1. [PDF Generator](#pdf-generator)

### Other Resources

1. [Import and Migrate](Runbook.migrate.md)
1. [Manual Data Modification](Runbook.ManualDataModifications.md)
1. [Bulk Delete Exams](Runbook.BulkDeleteExams.md)
1. [Embargo](Runbook.embargo.md)
1. [Language Support](Runbook.language.md)
1. [Data Specifications](Runbook.data.md)

<a name="common"></a>
## Common Service Conventions
All the applications in the RDW share common conventions and behaviors. This consistency across services makes maintaining the system less prone to error.

* **Dockerized**. They have been built to run in containers managed by an orchestration framework like Kubernetes.  
* **Java 8**. (non-issue since JDK is part of the docker image?)
* **Spring Boot**. 
    * [Configuration][1] and [Common Properties][2]. 
    * [Actuator end-points][3]. 
    * Logging. 

#### Configuration
As [Spring Boot][1] applications, the configuration settings for all applications come from:

* **Built-in defaults**. All applications have built-in defaults for everything except secrets (e.g. credentials).
* **Environment variables**. These may be used to override any default setting. However, they are used primarily to configure the environment profile and configuration server settings.
* **Command Line Options**. These may be used to override any default setting. In a container orchestration framework, these are seldom used.
* **Configuration server**. There is a central configuration server that all applications use to get environment-specific settings. Properties served up by the configuration server may be encrypted, protecting environment secrets. 

There are settings that all the applications use to bootstrap to the configuration server. These are generally set using environment variables in the orchestration framework, for example: 

```bash
CONFIG_SERVICE_ENABLED=true
CONFIG_SERVICE_LABEL=master
CONFIG_SERVICE_URL=http://configuration-service
```

Secrets in the configuration files may be encrypted to further protect them. The configuration service automatically decrypts these values when providing them to a service. To encrypt a value, pass it to the configuration service `encrypt` end-point. Using curl this looks like: `curl -X POST --data-urlencode "my+secret" http://localhost:8888/encrypt`. The encrypted value is then prefixed with `{cipher}` and wrapped in quotes, for example:
```yaml
spring:
  datasource:
    password: '{cipher}5e0e421375abd307f87f07a8ed4dab5ee9f105e8d4845ecf037f7ebdaeaf5709' 
``` 

The configuration server can combine multiple configuration files for a service. One use of this feature is to have common settings in one file, as shown in this [Annotated Configuration](../config/application.yml).


<a name="import-service"></a>
## Import Service
The import service is the REST end-point for submitting data to the system. It is responsible for archiving all imported data and then passing the work, via message queue, to payload processors. It uses OAuth2 for client validation. It is horizontally scalable for HA and overall throughput. A single process can handle a few dozen clients with an average latency of 200-300ms per request. 

![Import Service](import-service.png)

#### Configuration
The [Annotated Configuration](../config/rdw-ingest-import-service.yml) describes the properties and their effects.


<a name="package-processor"></a>
## Package Processor
The package processor processes assessment packages, organizations and accommodations submitted to the system. It is responsible for parsing and validating the data before writing it to the data warehouse. Due to infrequent demand this processor has not been designed for high concurrency and only a single instance should be run.

![Package Processor](package-processor.png)

#### Configuration
The [Annotated Configuration](../config/rdw-ingest-package-processor.yml) describes the properties and their effects.


<a name="exam-processor"></a>
## Exam Processor
This processor handles parsing, validating and writing test results to the data warehouse. It also extracts student information from the test results, creating and updating them as necessary. It is horizontally scalable with each process handling 20-30 exams/sec.

![Exam Processor](exam-processor.png)

#### Configuration
The [Annotated Configuration](../config/rdw-ingest-exam-processor.yml) describes the properties and their effects.


<a name="group-processor"></a>
## Group Processor
This processor handles parsing, validating and writing student group information to the data warehouse. It is horizontally scalable but a single process can handle a large volume of data (because most of the work is being done by the database itself), and group changes are relatively infrequent. Unlike other processors it must access the archive store directly (instead of getting its data from the message queue). It also uses Aurora's native ability to load directly from S3 to avoid copying data excessively.

![Group Processor](group-processor.png)

#### Configuration
The [Annotated Configuration](../config/rdw-ingest-group-processor.yml) describes the properties and their effects.


<a name="migrate-reporting"></a>
## Migrate Reporting
The migrate reporting service moves data from the warehouse to the reporting database. The service is not built to be horizontally scalable. Having more than one migrate reporting process will result in unpredictable behavior. 

Data is migrated based on import status (PROCESSED) and created/updated timestamps. The migration process is scheduled to run periodically. In each period, data is processed in small batches until there is no full batch remaining. The migration occurs in two steps, first from warehouse to staging, then from staging to reporting (the staging tables exist in the reporting database).

![Migrate Reporting](migrate-reporting.png)

The migrate service is controlled by two conditions: the user-controlled run state and the system-generated enabled state. The status end-points can be used to see the current status and pause/resume the service. Please refer to [Troubleshooting Migrate](Troubleshooting.md#migrate) for more details.

#### Configuration
The [Annotated Configuration](../config/rdw-ingest-migrate-reporting.yml) describes the properties and their effects.
 

<a name="migrate-olap"></a>
## Migrate OLAP
The migrate OLAP service is responsible for migrating data from the data warehouse to the aggregate reporting OLAP data store. The service is not built to be horizontally scalable. Having more than one migrate olap process will result in unpredictable behavior.

Data is migrated based on import status (PROCESSED) and created/updated timestamps. The migration process is scheduled to run daily. Each time data is processed in large batches until there is no full batch remaining. The migration occurs in two steps, first from warehouse to staging, then from staging to olap (the staging tables exist in the olap database). The migration tables exist in a separate, non-OLAP database.

![Migrate OLAP](migrate-olap.png)

The migrate service is controlled by two conditions: the user-controlled run state and the system-generated enabled state. The status end-points can be used to see the current status and pause/resume the service.

#### Configuration
The [Annotated Configuration](../config/rdw-ingest-migrate-olap.yml) describes the properties and their effects.
 

<a name="taks-service"></a>
## Task Service
This service is responsible for executing scheduled tasks. Currently this includes:
* Synchronizing organization data from ART (daily).
* Generating an import reconciliation report (daily).
* Resubmitting unprocessed test results (daily).

Only a single instance should be run since the task execution uses a simple, uncoordinated, time-based strategy.

![Task Service](task-service.png)

#### Configuration
The [Annotated Configuration](../config/rdw-ingest-task-service.yml) describes the properties and their effects.


<a name="reporting-webapp"></a>
## Reporting Web App
This is the main reporting web application used by customers. It provides the UI experience for all users, including functionality like managing student groups (which used to be in a separate admin webapp). It is horizontally scalable with each process handling 2000-3000 concurrent users (this deployment is expected to have up to 15000 concurrent users).

The reporting web app is a UI-only application that handles some security (SSO redirects) and the presentation of data. All the "heavy lifting" of querying data is handled by the API services.

![Reporting Web App](reporting-webapp.png)

#### Configuration
The [Annotated Configuration](../config/rdw-reporting-webapp.yml) describes the properties and their effects.


<a name="reporting-service"></a>
## Reporting Service
This service provides the back-end API for reports against the reporting data mart, i.e. individual test results. It is horizontally scalable and many instances should be run to deal with report requests from the webapp.

![Reporting Service](reporting-service.png)

#### Configuration
The [Annotated Configuration](../config/rdw-reporting-service.yml) describes the properties and their effects.


<a name="aggregate-service"></a>
## Aggregate Service
This service provides the back-end API for reports against the OLAP data store, i.e. aggregate reports. It is horizontally scalable; however, the OLAP data store is typically the bottleneck and running many instances of this service may not result in better overall throughput.

![Aggregate Service](aggregate-service.png)

#### Configuration
The [Annotated Configuration](../config/rdw-reporting-aggregate-service.yml) describes the properties and their effects.


<a name="admin-service"></a>
## Admin Service
This service provides the back-end API for administrative functionality including management of student groups, instructional resource links, embargo settings. It is horizontally scalable, however administrative management is limited in scope and a single instance is usually sufficient.

![Admin Service](admin-service.png)

#### Configuration
The [Annotated Configuration](../config/rdw-reporting-admin-service.yml) describes the properties and their effects.


<a name="report-processor"></a>
## Report Processor
This processor generates PDF reports. It is horizontally scalable and many instances should be run to deal with reporting load.

![Report Processor](report-processor.png)

#### Configuration
The [Annotated Configuration](../config/rdw-reporting-report-processor.yml) describes the properties and their effects.


<a name="reporting-webapp"></a>
## PDF Generator
This application converts HTML to PDF. It is used by the report processor. It is horizontally scalable and many instances should be run to deal with reporting load. Note the PDF generator is not a Spring Boot application: it doesn't use the central configuration server, it doesn't have the same actuator end-points, and logging is different.

![PDF Generator](pdf-generator.png)

#### Configuration
There are no configuration options for the PDF generator.


---
[1]: https://docs.spring.io/spring-boot/docs/1.5.2.RELEASE/reference/htmlsingle/#boot-features-external-config
[2]: https://docs.spring.io/spring-boot/docs/1.5.2.RELEASE/reference/htmlsingle/#common-application-properties
[3]: https://docs.spring.io/spring-boot/docs/1.5.2.RELEASE/reference/htmlsingle/#production-ready-endpoints
 

			 
