## Monitoring

**NOTE: please avoid putting environment-specific details _especially secrets and sensitive information_ in this document.**

**Intended Audience**: this document provides information for monitoring the Reporting Data Warehouse. Operations and system administrators will find it useful.

Monitoring RDW applications includes monitoring:

* Database. 
* Logging.
* Application Status.

### Database 
There are a number of tables that provide useful information about the state of the system.

#### Import Status
As data is accepted into the system an import record is created. Once the data is processed the status of the import record is updated to reflect success or a number of different error conditions. Monitoring the import table will catch any such issues. A query against the warehouse that counts all failures:

```sql
SELECT s.name status,  i.count
FROM
  (SELECT status, count(*) count FROM import WHERE content = 1 AND status < 0   GROUP BY status) i
  JOIN import_status s ON s.id = i.status
UNION
SELECT '--------------', '-----'
UNION
SELECT 'TOTAL', count(*)
FROM import
WHERE content = 1 AND status < 0
```
A more refined query that shows similar information for just the last day:

```sql
SELECT
  count(*) count,
  i.status,
  s.name AS status_name,
  cast(i.updated AS DATE) date
FROM import i
   JOIN import_status s ON i.status = s.id
WHERE 
  i.updated >= (CURRENT_DATE - INTERVAL 1 DAY) and i.updated < CURRENT_DATE -- or a specific date '2017-07-04'
  AND status < 0
GROUP by i.status, status_name, cast(i.updated AS DATE);
```

Another query to show more information about the failures:

```sql
SELECT
  i.id,
  creator,
  created,
  updated,
  s.name AS status,
  message
FROM import i
  JOIN import_status s ON s.id = i.status
WHERE i.content = 1 AND i.status < 0 AND s.name IN
  ('BAD_DATA',
   'BAD_FORMAT',
   'INVALID',
   'UNAUTHORIZED',
   'UNKNOWN_ASMT',
   'UNKNOWN_SCHOOL')
ORDER BY created DESC
```
The above query can be further refined to keep only interesting statuses.

For any of the queries, a non-empty result set indicates that there is unprocessed data in the system. Refer to 
[Troubleshooting][1] to resolve issues.

#### Monitor Ingest Speed
A new ingest request is captured by the ACCEPTED status of the import. Once the data is loaded into the warehouse the status is updated accordingly. Each ingest is different and hence the processing time will vary, but in general it is expected to take less than a minute.

To monitor for slow imports:

```sql
SELECT count(*) FROM import WHERE status = 0 AND updated > (CURRENT_TIMESTAMP + INTERVAL 60 SECOND);
```
If there are slow imports please refer to [Troubleshooting][1] to resolve. Although not urgent, this will affect the timeliness of the reporting data.

#### Monitor Migrate Failures
The migrate process is managed by the “migrate-reporting” service. The service records its status into the table. For all possible migrate statuses please refer to `migrate_status` table:

```sql
select * from migrate_status;
```

The service will suspend itself if there is a failure. To check for the failure, run:

```sql
SELECT * FROM migrate WHERE status = -20;
```
If there are failures refer to [Troubleshooting][1] to resolve. 

**This condition requires immediate attention since new test results will not be visible in the reporting application until it is resolved**.
 
#### Monitor Migrate Speed
The  “migrate-reporting” service continuously migrates newly imported data from `warehouse` to `reporting`. The data is moved in batches defined by the `migrate`'s `first_at` and `last_at` timestamps. Each batch is different and hence the processing time will vary, but in general it is expected to take less than a minute.

To establish an average speed of the migrate for a particular installation, check the processing speed of the successful migrates on any given day:

```sql
SELECT timestampdiff(SECOND, created, updated) runtime_in_sec
FROM migrate
WHERE status = 20 AND
      created >= (CURRENT_DATE - INTERVAL 1 DAY) AND created < CURRENT_DATE; -- or a specific date '2017-07-04'
```

Or check the average processing time over time:

```sql
SELECT avg(timestampdiff(SECOND, created, updated)) avg_runtime_in_sec
FROM migrate
WHERE status = 20 AND created >= '2017-07-04' AND created < '2017-12-04'; -- substitute dates with your values
```

To monitor the top 5 slowest successful migrates on a day before any given day run the following:

```sql
SELECT timestampdiff(SECOND, created, updated) runtime_in_sec
 FROM migrate
 WHERE status = 20 AND
       created >= (CURRENT_DATE - INTERVAL 1 DAY) AND created < CURRENT_DATE
 ORDER BY runtime_in_sec DESC
 LIMIT 5;
```  

If migrates are taking longer than expected, or if the monitoring shows a consistent increase in run time over time the system is degrading. Refer to [Troubleshooting][1] for instructions on diagnosing the situation. Although not urgent, this will affect the timeliness of the reporting data.

### Logging

#### Log Collection
The approach is to use fluentd on the nodes to tail logs and forward entries to a central service like Elasticsearch. If you ask six developers how to install and configure fluentd in a kubernetes cluster you'll get between nine and twelve different answers. If you want to do a little reading:

* [Overview of Kubernetes logging](https://kubernetes.io/docs/concepts/cluster-administration/logging/)
* [Kubernetes logging with Elasticsearch/Kibana](https://kubernetes.io/docs/tasks/debug-application-cluster/logging-elasticsearch-kibana/)
* [Kubernetes logging with Fluentd](https://docs.fluentd.org/v0.12/articles/kubernetes-fluentd)
* [Logging to AWS Elasticsearch from Kubernetes](https://medium.com/@while1eq1/logging-to-aws-elasticsearch-service-from-kubernetes-855ad0959251)

What we did:

1. Create an AWS Elasticsearch instance. From the AWS Elasticsearch Service dashboard, `Create a new domain`:
    * Name: rdw-qa
    * Latest version (5.5)
    * Instance: ts.small.elasticsearch (1)
    * Storage: EBS 30GB (for now)
    * Public (for now)
1. Set access policy to allow full control from within VPC. For now, allow access from the interwebs too:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "*"
      },
      "Action": "es:*",
      "Resource": "arn:aws:es:us-west-2:269146879732:domain/rdw-qa/*",
      "Condition": {
        "IpAddress": {
          "aws:SourceIp": "172.31.0.0/16"
        }
      }
    },
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "*"
      },
      "Action": "es:*",
      "Resource": "arn:aws:es:us-west-2:269146879732:domain/rdw-qa/*",
      "Condition": {
        "IpAddress": {
          "aws:SourceIp": [
            "0.0.0.0/0"
          ]
        }
      }
    }
  ]
}
```
1. Note ES settings:
    * Endpoint: https://search-rdw-qa-ocdbui57wpdryyx25hhvlh44fe.us-west-2.es.amazonaws.com
    * Domain ARN: arn:aws:es:us-west-2:269146879732:domain/rdw-qa
    * Kibana: https://search-rdw-qa-ocdbui57wpdryyx25hhvlh44fe.us-west-2.es.amazonaws.com/_plugin/kibana/
1. Test:
```bash
$ curl https://search-rdw-qa-ocdbui57wpdryyx25hhvlh44fe.us-west-2.es.amazonaws.com
{
  "name" : "YbZyaKb",
  "cluster_name" : "269146879732:rdw-qa",
  "cluster_uuid" : "1OOuXtMvQCGkKAYhgGhHKQ",
  "version" : {
    "number" : "5.5.2",
    "build_hash" : "b2f0c09",
    "build_date" : "2017-08-21T22:31:55.076Z",
    "build_snapshot" : false,
    "lucene_version" : "6.6.0"
  },
  "tagline" : "You Know, for Search"
}
```
1. Deploy fluentd agents to collect logs from each node.
```bash
kubectl apply -f fluentd.yml
```
1. TODO - how to access kibana without opening it to the whole world

    
#### Log Messages
These are log messages that can be used to trigger alerts about rare but expected issues. In general, all ERROR 
messages should trigger alerts, and messages at INFO, DEBUG, TRACE levels may be safely ignored. This section is
about WARN messages, and exceptions to the general handling. 

##### Exam Processing
Any failure while processing an exam (aka test result, aka TRT) will produce a log message. The text will vary depending on the error, but it will look like: 
```text
2017-09-24 03:58:04.124  WARN 1 --- [nio-8080-exec-9] o.o.r.i.p.ExamProcessorConfiguration  : failed with an ...
```
It is important to understand and resolve these failures since the test result will not appear in reports.

##### Migrate Disable
The migrate process will disable itself if it encounters problem. The log entry looks like this:
```text
2017-09-24 03:58:04.124  WARN 1 --- [nio-8080-exec-9] o.o.r.m.c.MigrateJobRunner  : disabling scheduled migrate jobs; manually fix last migrate status to enable
```
See [Troubleshooting](Troubleshooting.md#migrate) for details on the migrate service.

##### Update Organizations
The update-organizations task can encounter a number of problems. Because these failures are not immediately obvious monitoring these warnings is recommended:
```text
2017-09-24 03:58:04.124  WARN 1 --- [nio-8080-exec-9] o.o.r.i.t.r.ArtClient                  : Unexpected response status from ART:404
...
2017-09-24 03:58:04.124  WARN 1 --- [nio-8080-exec-9] o.o.r.i.t.s.i.RestImportServiceClient  : organization import not accepted, id: 123, status: BAD_DATA
```
If desired successful organization posts can be monitored by watching for:
```text
2017-09-24 03:58:04.124  INFO 1 --- [nio-8080-exec-9] o.o.r.i.t.s.i.RestImportServiceClient  : posted organizations import, id: 123, status: ACCEPTED
```

##### Reconciliation Report
The reconciliation report task can fail either completely or with partial delivery. Messages will look like:
```text
2017-09-24 03:58:04.124  WARN 1 --- [nio-8080-exec-9] o.o.r.i.t.s.i.DefaultReconciliationService  : Failed to send reconciliation report to ArchiveReportSender: ...
...
2017-09-24 03:58:04.124  WARN 1 --- [nio-8080-exec-9] o.o.r.i.t.ReconciliationReportConfiguration : Error sending reconciliation report ...
``` 

##### Duplicate Payload
The import service calculates a hash for any payload submitted and will ignore duplicates while logging a message at INFO level. Although this is harmless from a system point of view, excessive occurrences could indicate bad data flow, especially for EXAMs. Note that there is an update-organization task that runs daily and will usually produce this log message since the organization data in ART doesn't change regularly.  
```text
2017-09-24 03:58:04.124  INFO 1 --- [nio-8080-exec-9] o.o.r.i.service.DefaultImportService     : Ignoring EXAM payload with existing digest 498B7E8EB17D624C2C6C5829B9664576
...
2017-09-24 04:00:06.797  INFO 1 --- [nio-8080-exec-9] o.o.r.i.service.DefaultImportService     : Ignoring ORGANIZATION payload with existing digest EDD59FFE5432D9234177B2770B0A2FAB
```

#### Red Herrings
There are some log messages that look scary but are not. Because they are emitted by 3rd party libraries we won't clean them up but we will note some of them here. 

##### AmazonS3 Failure with Retry
The following info message may appear in the log followed by a stack trace. Although stack traces are normally an indicator of Bad Things, in this case the system will retry and get past the problem. The exact stack trace depends on the circumstances but it will be one of the following:
```text
2017-10-12 21:25:54.836  INFO 41018 --- [pool-2-thread-1] com.amazonaws.http.AmazonHttpClient      : Unable to execute HTTP request: The target server failed to respond

org.apache.http.NoHttpResponseException: The target server failed to respond ...
org.apache.http.conn.ConnectTimeoutException: Connect to ...
```

##### Load Balancer HTTP/HTTPS
The following info message may appear in the log followed by a stack trace. Although stack traces are normally an indicator of Bad Things, in this case it is not a problem. 
```text
2017-10-27 14:39:52.458  INFO 1 --- [nio-8080-exec-4] o.apache.coyote.http11.Http11Processor   : Error parsing HTTP request header
 Note: further occurrences of HTTP header parsing errors will be logged at DEBUG level.

java.lang.IllegalArgumentException: Invalid character found in the HTTP protocol
...
``` 

### Application Status
The applications present the Spring Boot Actuator endpoints as well as the SmarterBalanced diagnostic status endpoint. These are served on a non-standard port (8008) and are not exposed outside the VPC. 


[1]: ./Troubleshooting.md