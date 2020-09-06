# AWS Dev Associate Notes #

## Elastic Beanstalk ##

- It is possible to set up xray by adding an xray-demon.config to the .ebextensions directory
- EB can be used for long running tasks. EB uses SQS to achieve this
- Immutable deploy method prevents original deployment from being overwritten. 
- You can add a cron.yml to your source bundle
- Elastic Beanstalk has its own CLI tool - `eb deploy`
- If you deploy RDS with Elastic Beanstalk and you want to retain the database when deleting the EB environment you must
  - Perform a blue/green deployment
  - Take a snapshot of the database
  - Attach the snapshot to a new RDS
  - Remove the security group
- When deleting environment EB does not retain automatic backups
- EB does not make automated changes to IAM roles

## Lambda ##

- Use parameter store for shared connection strings
- Can be invoked sync or async. Use event invocation type to make async
- InvokeAsync() is deprecated
- Store files in /tmp
- Use execution context for db connections
- Traffic shifting / lambda aliases are used to perform quick rollback
- There is no half-at-a-time deployment option
- There is no in-place deployment option
- Canary is the quikcest deployment option when you don't want to shift all at once
- Concurrent executions = (invocations per second) x (average execution duration in seconds)
- Default concurrent exection maximum is 1000 per account. Can be increased with a support ticket
- You can set 900 concurrent executions - 100 is always left unreservable
- If you have a function that has 450 reserved concurrent executions and another function with none and the second one is being throttled, you may need to reduce concurrent executions on the first one
- Must return json to API gateway - xml not supported 
- Layers are used for reusable parts of code like frameworks. They reduce the size of the deployable
- Lambda CPU is increased by increasing memory
- If deploying lambda with CloudFormation you can include the code inline in the `ZipFile` section
- Code is max 250mb uncompressed
- Code is max 50mb compressed
- Min memory is 128mb
- Max memory is 3008mb
- Max cpu is 2 vCPUs
- Lambda can be tested locally with SAM CLI
- If creating via CLI and you get the error `InvalidParameterValueException` it most likely means a bad IAM role was passed

## SWF / Step Functions ##

- Step functions is fully managed
- Workflow is managed but infra is up to you
- Use the Task step type to create an event
- Use the activity step type to wait for N seconds
- The Wait step type pauses the state machine
- Markers are used to count things

## S3 ##

- To allow unauthenticated access to some parts of a bucket, use a Cognito Identity Pool
- Use a VPC Endpoint to allow VPC access to a bucket
- If querying lots of keys using the CLI, reduce it using `--page-size and` `--max-items`
- To allow customers provide their own encryption keys use these headers:
  - x-amz-server-side-encryption-customer-algorithm
  - x-amz-server-side-encryption-customer-key
  - x-amz-server-side-encryption-customer-key-MD5
- To use default AWS KMS encryption:
  - x-amz-server-side-encryption
- Transfer Accelration makes inter-region transfers faster
- Both buckets must be versioned to enable cross region transfer
- If multipart uploading a large file that requires a KMS key, the `kms:decrypt` policy must be present

## DynamoDB ##

- Sererless NoSQL database
- Uses partition and sort key
- Local indexes can only be created when creating a table
- Local indexes support eventual and strong consistency
- Global secondary indexes can be created after
- Eventual consistency only
- Can `Scan` or `Query`
- Scan goes over all partitions
- Can reduce page sizes to improve performance
- DAX also increases performance at a cost - sub microsecond latency
- Can use TTLs on data that doesn't need to stick around
- Can use DynamoDB streams to stream changes. AKA triggers
- Conditional writes can be used where multiple users are attempting to write an item at the same time
- Use `INDEXES` when calling `ReturnConsumedCapacity` to get WCU of table and indexes
- Use `TOTAL` to get just the table WCU
- Can use atomic counters for counting stuff - it is accurate
- Hot partition, steps to fix:
  - Distribute reads and writes evenly among partitions
  - Error retries and exponential backoff
  - Increase read or write capacity
  - Implement DAX
- Store data in hierarchial format rather than in multiple tables
- Encrypts data at rest by default
- Can have a maximum of 5 local / global secondary indexes on a table

## SNS ##

- Can filter using subscription filters
- Push - no polling

## X-Ray ##

- In order to filter documents, add custom attributes as annotation to segment document, then use filter expressions on console
- Better than CloudWatch for diagnosing multiple systems
- To enable on lambda, use environment variables:
  - `AWS_XRAY_CONTEXT_MISSING`
  - `_X_AMXN_TRACE_ID`
- Use subsegments to record downstream calls such as to AWS services
- Annotations can be searched
- Metadata are key-value pairs that aren't indexed - data that doesn't need to be searched
- X-Ray API: use `GetTraceSummaries` to get IDs and `BatchGetTraces` to get full trace data
- X-Ray demon listens on UDP port 2000 and relays data to X-Ray API
- X-Ray only records a statistically significant number of records

## API Gateway ##

- Caching can be enabled
- Caching can be bypassed & invalidated by passing the header `cache-control: max-age-0`
- When passing requests to a non lambda endpoint where you need to map the request, use HTTP
- 504 means timeout
- Enforce request structure (i.e. query string must contain `x`) by changing method request in API Gateway

## CloudFront ##

- Use Lambda@Edge and Cognito to ensure unauthenticated users can't access a resource
- For end to end SSL, configure the Viewer Policy and Origin Policy
- If getting 504 errors:
  - Create a Failover Origin
  - Use Lambda@Edge to do authentication

## Kinesis ##

- Shards should be equal to workers
- If scaling in/out, scale both shards and workers
- Merge cold shards or split cold shards to scale in / out
- Stream records are available for 24 hours
- When using lambda, the number of shards is the maximum concurrent executions. Lambda processes each shard's message in sequence
- Each shard can write 1mbps
- Each shard can read 2mbps

## RDS ##

- Slow query log can be enabled to identify slow queries
- Read replicas can quickly improve read performance. Simpler to implement than caching
- If deleting the RDS instance and you need to retain the data use the `SNAPSHOT` deletion option
- Can enable enchanced monitoring to get CPU metrics
- Transparent Data Encryption is used to encrypt MSSQL data at rest

## SAM ##

- `sam package` and `cloudformation package` are identical
- `AWS::Serverless::Application` is the type to use when creating a serverless application
- SAM stuff goes in the `Transform` section of CloudFormation
- Not something you use to deploy

## ElastiCache ##

- Memcached simple, Redis feature rich
- Redis mostly single threaded
- Redis can be replicated for high availability
- WriteThrough is a method to keep the cache up to date
- Redis supports up to 250 nodes in a cluster

## Security ##

- Access keys are used to get into AWS from the outside
- Cross account access is possible
- IAM Policy Simulator is useful to simualate policy changes before committing to them
- AssumeRoleWithSAML provides an AccessKeyId, Secret Access Key and Security Token
- STS:
  - Identity providers must be compatable with OpenID or SAML 2.0
  - Valid providers:
    - IAM from another AWS account
    - Web IdP
    - Active Directory
  - If using MFA, use GetSessionsToken
  - Create a custom identity broker when integrating LDAP with VPC
  - OpenId supports Id, Access and Refresh tokens
- Evelope Encryption: encrypt plaintext data with a data key and then encrypt the data key with a top-level plaintext master key
- Control plane is used for managing compute resources
- Data plane is used for managing applications
- Control plane uses APIs
- Control plane and data plane are combined in serverless services like DynamoDB
- Use WAF to protect against XSS / SQL injection

## Alarms ##

- Evaluation period is number of datapoints to check before alarm
- Period is the interval to check the metric

## SQS ##

- Long polling is appropriate for when a consumer takes a long time to produce messages
- Used if `WaitTimeSeconds` > 0
- A Dedupe ID can be added by the producer. Messages with same Dedupe IDs will be dropped if they appear within 5 minutes of each other
- FIFO queues deliver exactly once
- `ReceiveMessageWaitTimeSeconds` lengthens polling
- `VisibilityTimeout` is a property on a message telling SQS how long to make it invisible before other consumers can pick it up

## Cognito ##

- Identity pools are used to grant users access to your AWS resources
- Cognito sync is for syncing user data between devices
- App sync is for syncing shared data between devices

## Load balancing ##

- `X-Forwarded-For` header contains the IP of the caller

## EC2 ## 

- EC2 Spot and DynamoDB are considered very elastic
- If using spot instances, make use of termination notice and SQS for splitting workloads
- To get more metrics, install CloudWatch agent
- CloudWatch doesn't monitor memory, swap or disk space. Install the agent and use scripts to achieve this


## EBS / EFS ##

- To attach EBS:
  - format it
  - mount it
- To detach EBS:
  - stop instance
  - detach
- EBS can be mounted to one instance
- EBS Snapshots cost money
- EFS can be mounted to many instances

## CodeDeploy ##

- Can deploy to on prem

## VPC ##

- A subnet can only be associated with one route table at a time

## CloudFormation ##

- Stacksets can be used to deploy stacks across different accounts
- Use `ClientRequestToken` to correlate with CloudTrail
- To get around size limit, upload to S3
- OpsWorks: use cfn-init to dynamically install packages, create files, start services
- OpsWorks: can handle chef and puppet
- OpsWorks: something is called on all instances when one comes up

## CodeStar ##

- Is a tool for monitoring app activity and day to day tasks

## RedShift ##

- An OLAP database
- Queries executed on leader nodes

## CloudHSM ##

- Hardware security

## Route53 ##

- Routing types:
  - Simple
  - Failover
  - Geolocation
  - Geoproximity
  - Latency
  - Multivalue
  - Weighed routing

## Billing ##

- Cost explorer for exploring costs
- Cost explorer has an API
- Trusted advisor for realtime advice on provisioning resources
- Usage report for detailed info

## CloudTrail ##

- Logs to S3
- Can be streamed to CloudWatch and metric filters can be used to set up alarms

## WorkDocs ##

- Auth with SimpleAD

## CodePipeline ##

- Trigger lambdas using CloudWatch events

## Data Migration ##

- Snowmobile is literally a truck full of data, support for many PB
- Snowball is a little box which can support a few PB
- S3 not suitable for large transfers

## Polly ##

- TTS service
- Not cross region
