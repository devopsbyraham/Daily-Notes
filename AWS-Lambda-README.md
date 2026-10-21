# AWS Lambda --- Student Notes and Practical Labs

## 1. What is AWS Lambda?

**AWS Lambda is a serverless compute service that runs code in response
to events without requiring you to provision or manage the underlying
servers.**

Memory line: **EC2 = you manage the server; Lambda = AWS manages the
execution infrastructure and you focus on the code.** Serverless does
not mean there are no servers; AWS manages them for you.

### Real-world use cases

-   S3 upload → process images or documents.
-   API Gateway → Lambda → backend API logic.
-   EventBridge Scheduler → start/stop approved EC2 instances.
-   SQS/SNS → process queued jobs or notifications.
-   CloudWatch alarms → send alerts or run approved remediation.
-   DynamoDB Streams/Kinesis → process supported record streams.

## 2. With Lambda vs. without Lambda

Example: stop development EC2 instances at 8 PM.

  -----------------------------------------------------------------------
  Without Lambda                      With Lambda
  ----------------------------------- -----------------------------------
  Maintain a server/script host       Run a function on demand or on a
                                      schedule

  Patch and monitor that host         AWS manages Lambda execution
                                      infrastructure

  Host may stay running while idle    No continuously running server is
                                      needed for the function itself

  More operational overhead for small Often less operational overhead
  automation                          

  Host can incur idle costs           Lambda charges depend on requests
                                      and duration; related services can
                                      also cost money
  -----------------------------------------------------------------------

Lambda is not always cheaper or better. Persistent services,
long-running jobs, or custom OS requirements may fit EC2 or containers
better.

## 3. Lambda vs. EC2

  -----------------------------------------------------------------------
  Feature                 Lambda                  EC2
  ----------------------- ----------------------- -----------------------
  Infrastructure          AWS manages execution   You manage instance
                          infrastructure          OS/configuration

  Execution               Event-driven function   Virtual machine; can
                                                  run persistent services

  Scaling                 Managed concurrency     Configure scaling,
                          scaling, subject to     e.g. Auto Scaling
                          quotas                  

  Billing                 Requests and execution  Instance runtime and
                          duration, plus related  other resources
                          charges                 

  Maximum execution       15 minutes per          Depends on application
                          invocation              and instance lifecycle

  Best fit                Short-lived             Persistent or
                          event-driven tasks      long-running workloads
  -----------------------------------------------------------------------

## 4. Key terminology

-   **Function:** Code and configuration Lambda executes.
-   **Handler:** Entry point Lambda calls, e.g. `lambda_handler`.
-   **Event:** Input describing what happened or what was requested.
-   **Context:** Runtime metadata such as request ID and remaining
    execution time.
-   **Trigger/event source:** Mechanism that causes invocation.
-   **Execution role:** IAM role that grants the function access to AWS
    APIs.
-   **Invocation:** Request to execute the function.
-   **Concurrency:** Number of invocations running simultaneously.
-   **Timeout:** Maximum allowed duration for one invocation.
-   **Environment variables:** Runtime configuration values.
-   **Version:** Immutable snapshot of code and most configuration.
-   **Alias:** Named pointer to a published version, such as `dev`,
    `staging`, or `prod`.

## 5. Architecture

``` text
Browser / App → API Gateway → Lambda → DynamoDB / S3 / other services

EventBridge Scheduler → Lambda → EC2 API → Start/stop approved instances
```

## 6. Triggers and invocation types

  -----------------------------------------------------------------------
  Trigger/source                      Common use
  ----------------------------------- -----------------------------------
  API Gateway                         HTTP request invokes backend code

  S3                                  Process an uploaded object

  EventBridge Scheduler               Run on a schedule

  EventBridge event bus               React to AWS/custom events

  SQS                                 Process queued messages

  SNS                                 Process notifications

  DynamoDB Streams                    React to table changes

  Kinesis Data Streams                Process stream records

  Direct invocation                   SDK, CLI, console, or another
                                      service

  Application Load Balancer           Invoke Lambda in supported
                                      configurations
  -----------------------------------------------------------------------

A **trigger** is the mechanism; the **event** is the input payload.

-   **Synchronous:** Caller waits for a response; common with API
    Gateway and direct SDK calls.
-   **Asynchronous:** Lambda accepts an event for later processing;
    caller does not wait for the final result. Common examples include
    S3/EventBridge integrations.
-   **Poll-based:** Lambda polls a source and retrieves batches,
    e.g. SQS, Kinesis, and DynamoDB Streams event source mappings.

Retry, ordering, batching, and failure behavior varies by integration.
Configure failure handling and monitoring.

### Idempotency

Events may be delivered more than once. Make handlers **idempotent**:
processing the same event twice should not create an unintended
duplicate effect. Use stable event IDs and conditional writes where
appropriate.

## 7. Cold start vs. warm start

**Cold start:** Lambda prepares a new execution environment. This may
include runtime setup, loading dependencies, and initialization code.

**Warm start:** Lambda reuses an existing execution environment, so
initialization may already be complete.

  -----------------------------------------------------------------------
  Cold start                          Warm start
  ----------------------------------- -----------------------------------
  New environment is prepared         Existing environment is reused

  May add initialization latency      Usually less initialization
                                      overhead

  Happens when Lambda needs new       Happens when a suitable environment
  environments                        is reused
  -----------------------------------------------------------------------

Warm reuse is **not guaranteed**. Do not store essential state only in
memory or assume an environment persists.

### Reduce cold-start impact

1.  Keep dependencies/package size reasonable.
2.  Avoid unnecessary module-level initialization.
3.  Initialize reusable SDK clients outside the handler when
    appropriate.
4.  Benchmark memory/runtime settings.
5.  Consider **Provisioned Concurrency** for latency-sensitive
    functions; it costs extra.
6.  Consider SnapStart only for supported runtimes/configurations.
7.  Measure actual latency before optimizing.

Provisioned Concurrency keeps a configured number of environments
initialized; it does not guarantee every invocation avoids startup
delay.

## 8. High availability and reliability

Lambda is a managed, highly available service within a Region. Your
whole application can still fail due to code, permissions, downstream
outages, quotas, networking, or bad deployments.

Best practices: - Use resilient configurations for databases and
dependencies. - Make handlers idempotent. - Configure retries and
failure destinations/DLQs where supported. - Monitor with CloudWatch
logs, metrics, alarms, and tracing. - Use concurrency controls to
protect downstream services. - Use versions and aliases for controlled
deployments and rollbacks. - Test throttling, timeouts, and partial
failures. - A Lambda function attached to a VPC does not automatically
gain internet access.

**Concurrency:** number of simultaneous invocations. - **Reserved
concurrency:** reserves capacity and sets a function's concurrency
limit; zero can stop new invocations. - **Provisioned concurrency:**
pre-initializes a configured number of environments. - **Throttling:**
invocations are limited by concurrency or quotas.

## 9. Lab 1 --- Create and test a Python Lambda

1.  AWS Console → **Lambda** → **Create function**.
2.  Choose **Author from scratch**.
3.  Name: `student-hello-lambda`.
4.  Choose a currently supported Python runtime.
5.  Create/select an execution role with basic CloudWatch Logs
    permissions.
6.  Create the function and replace starter code with:

``` python
import json

def lambda_handler(event, context):
    name = event.get("name", "Student")
    return {
        "statusCode": 200,
        "headers": {"Content-Type": "application/json"},
        "body": json.dumps({
            "message": f"Hello, {name}!",
            "request_id": context.aws_request_id
        })
    }
```

7.  Select **Deploy**.
8.  In **Test**, create this event and run it:

``` json
{"name": "AWS Student"}
```

9.  Inspect the result and open **Monitor → View CloudWatch logs**.

`statusCode` and `body` are commonly used with API Gateway proxy
integration. A direct Lambda test shows the returned object; it does not
publish a website.

## 10. Lab 2 --- Return an HTML CloudOps Dashboard

To serve HTML to a browser over HTTP, place **API Gateway** in front of
Lambda and configure the integration. A direct Lambda invocation does
not publish a public website.

1.  Create function `cloudops-dashboard` using a supported Python
    runtime.
2.  Ensure the execution role can write logs.
3.  Paste the following code, select **Deploy**, and test.
4.  For browser access, create an API Gateway HTTP API, integrate it
    with the function, and configure a route/stage.
5.  For production, add authentication and replace the static metrics
    with real monitoring data.

``` python
def lambda_handler(event, context):
    html = """<!DOCTYPE html>
<html>
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>CloudOps Dashboard</title>
<style>
body{font-family:Arial,sans-serif;background:#f4f7fb;margin:0;color:#172033}
header{background:#1f4fbf;color:#fff;padding:18px;text-align:center}
.container{max-width:1000px;margin:25px auto;padding:20px}
.cards{display:grid;grid-template-columns:repeat(auto-fit,minmax(200px,1fr));gap:18px}
.card{background:#fff;border-radius:10px;padding:20px;box-shadow:0 2px 10px rgba(0,0,0,.12)}
.big{font-size:34px;font-weight:bold;color:#1f4fbf}
.status{color:green;font-weight:bold}
footer{text-align:center;margin-top:25px;color:#666}
</style>
</head>
<body>
<header><h1>CloudOps Dashboard</h1></header>
<div class="container">
<div class="cards">
<div class="card"><h3>Running Services</h3><div class="big">18</div></div>
<div class="card"><h3>Lambda Invocations</h3><div class="big">24,892</div></div>
<div class="card"><h3>Application Status</h3><div class="status">Healthy</div></div>
</div>
<div class="card" style="margin-top:20px">
<h2>Deployment Summary</h2>
<p>Environment: <b>Demo</b></p>
<p>Region: <b>ap-south-1</b></p>
<p>Version: <b>v2.3.1</b></p>
<p>This sample page is returned by AWS Lambda.</p>
</div>
</div>
<footer>Sample AWS Lambda Web Application</footer>
</body>
</html>"""
    return {
        "statusCode": 200,
        "headers": {
            "Content-Type": "text/html; charset=utf-8",
            "Cache-Control": "no-store"
        },
        "body": html
    }
```

The displayed numbers are static sample values, not actual CloudWatch
metrics.

## 11. Lab 3 --- Launch EC2 from Lambda

**Cost/safety warning:** EC2 instances can incur charges. Use a lab
account, verify the Region and AMI, set a budget, and clean up
resources.

### IAM role

1.  IAM → **Roles** → **Create role**.
2.  Trusted entity: AWS service; use case: Lambda.
3.  Name it `StudentLambdaLaunchEC2Role`.
4.  Attach `AWSLambdaBasicExecutionRole` for logs.
5.  Add a custom policy for required EC2 actions and resources.
    `ec2:RunInstances` may require permissions on the image, instance,
    subnet, security group, network interface, and related resources;
    `ec2:CreateTags` may also be needed.
6.  Add `iam:PassRole` only if the EC2 instance needs an instance
    profile.
7.  Attach this role to the Lambda function under **Configuration →
    Permissions**.

Avoid broad administrator permissions in production. Have the lab
administrator validate the least-privilege policy for the chosen launch
configuration.

### Verify before running

-   AMI exists in the target Region.
-   Instance type is permitted.
-   Subnet and security group are suitable.
-   A key pair is only needed for access methods that use it.
-   Budget and cleanup plan are ready.

Your original values `ami-01a00762f46d584a1` and key pair `kprm` must be
verified in your account and in `ap-south-1`. AMI IDs are
Region-specific.

### Python code

Set `AMI_ID` in Lambda environment variables to a verified AMI ID in the
chosen Region.

``` python
import os
import boto3

REGION = os.environ.get("AWS_REGION", "ap-south-1")
AMI_ID = os.environ["AMI_ID"]
INSTANCE_TYPE = os.environ.get("INSTANCE_TYPE", "t3.micro")
ec2 = boto3.client("ec2", region_name=REGION)

def lambda_handler(event, context):
    response = ec2.run_instances(
        ImageId=AMI_ID,
        InstanceType=INSTANCE_TYPE,
        MinCount=1,
        MaxCount=1,
        TagSpecifications=[{
            "ResourceType": "instance",
            "Tags": [
                {"Key": "Name", "Value": "Lambda-Created-EC2"},
                {"Key": "CreatedBy", "Value": "StudentLambdaLab"},
                {"Key": "AutoCleanup", "Value": "true"}
            ]
        }]
    )
    instance_id = response["Instances"][0]["InstanceId"]
    return {
        "statusCode": 200,
        "body": f"Successfully requested EC2 launch: {instance_id}"
    }
```

This intentionally omits `KeyName`: a key pair is not mandatory for
every access method. If SSH is required, use a valid key pair and
restrict inbound access. Prefer Systems Manager Session Manager when
configured.

### Test

1.  Lambda → **Configuration → Environment variables** → add `AMI_ID`.
2.  Deploy code changes.
3.  Create test event `{}` and select **Test**.
4.  EC2 → **Instances**: verify the instance and tags.
5.  Check CloudWatch logs if it fails.
6.  Stop or terminate the instance when finished.

For production, allow-list AMIs, instance types, subnets, and security
groups; prevent duplicate events from launching duplicate instances.

## 12. Lab 4 --- Auto-stop EC2 with EventBridge Scheduler

### Permissions

Lambda execution role needs: - `AWSLambdaBasicExecutionRole`. -
`ec2:StopInstances` for intended instances. - Describe permissions to
find instances by tags.

Use least privilege instead of `AmazonEC2FullAccess` in production.

### Stop only tagged running instances

Tag the lab instances `AutoStop=true`.

``` python
import boto3

REGION = "ap-south-1"
ec2 = boto3.client("ec2", region_name=REGION)

def lambda_handler(event, context):
    response = ec2.describe_instances(Filters=[
        {"Name": "tag:AutoStop", "Values": ["true"]},
        {"Name": "instance-state-name", "Values": ["running"]}
    ])

    instance_ids = [
        instance["InstanceId"]
        for reservation in response["Reservations"]
        for instance in reservation["Instances"]
    ]

    if not instance_ids:
        print("No running instances found with AutoStop=true.")
        return {"stopped": []}

    ec2.stop_instances(InstanceIds=instance_ids)
    print(f"Requested stop for: {instance_ids}")
    return {"stopped": instance_ids}
```

### Create schedule

1.  Open **Amazon EventBridge** in the Lambda function's Region.
2.  Open **Scheduler** → **Create schedule**.
3.  Name it `stop-student-lab-instances`.
4.  Choose a recurring schedule, e.g. daily at 8:00 PM.
5.  Confirm the time zone. For India time, choose `Asia/Kolkata` if
    available.
6.  Target: **AWS Lambda Invoke**; select the stop function.
7.  Configure Scheduler's execution role so it can invoke only this
    function.
8.  Review schedule, target, role, and time zone; create it.
9.  Manually test with one tagged lab instance first.
10. Check CloudWatch logs and EC2 state after it runs.

EventBridge Scheduler and scheduled EventBridge rules are distinct
features. The Scheduler execution role that invokes Lambda is different
from the Lambda execution role that calls EC2.

### Start instead of stop

Tag intended instances `AutoStart=true` and use:

``` python
import boto3

REGION = "ap-south-1"
ec2 = boto3.client("ec2", region_name=REGION)

def lambda_handler(event, context):
    response = ec2.describe_instances(Filters=[
        {"Name": "tag:AutoStart", "Values": ["true"]},
        {"Name": "instance-state-name", "Values": ["stopped"]}
    ])

    instance_ids = [
        instance["InstanceId"]
        for reservation in response["Reservations"]
        for instance in reservation["Instances"]
    ]

    if not instance_ids:
        print("No stopped instances found with AutoStart=true.")
        return {"started": []}

    ec2.start_instances(InstanceIds=instance_ids)
    print(f"Requested start for: {instance_ids}")
    return {"started": instance_ids}
```

Stopping EC2 generally stops instance-compute charges, but EBS volumes,
snapshots, some Elastic IP configurations, and other resources may
continue to cost money.

## 13. Deployment workflow

1.  Create function.
2.  Choose supported runtime and architecture.
3.  Configure execution role.
4.  Write/upload code.
5.  Configure memory, timeout, environment variables, and networking if
    needed.
6.  Deploy.
7.  Create a test event and test.
8.  Configure trigger.
9.  Verify permissions, payload shape, and network paths.
10. Monitor logs, metrics, errors, throttles, and duration.

A successful console test does not prove the trigger, API route, IAM
permission, or production network path is correct.

## 14. Two IAM roles students must distinguish

-   **Lambda execution role:** grants the running code permission to
    call AWS APIs such as `ec2:StopInstances`; typically includes
    CloudWatch Logs permissions.
-   **EventBridge Scheduler execution role:** allows Scheduler to invoke
    the target Lambda function.

**Memory line:** Scheduler role = permission to invoke Lambda; Lambda
execution role = permission for code to access AWS services.

## 15. Monitoring and troubleshooting

Useful CloudWatch metrics: - `Invocations`: number of executions. -
`Errors`: failed invocations. - `Duration`: execution time. -
`Throttles`: throttled invocations. - `ConcurrentExecutions`:
simultaneous executions. - `IteratorAge`: useful for supported
stream-based sources.

  ------------------------------------------------------------------------------
  Symptom                   Likely cause                 What to check
  ------------------------- ---------------------------- -----------------------
  `AccessDenied`            Missing IAM/resource         Role, resource policy,
                            permission or explicit deny  boundary, SCP

  `InvalidAMIID.NotFound`   Wrong AMI or Region          Verify AMI in
                                                         configured Region

  `UnauthorizedOperation`   EC2 API not permitted        Execution role and
                                                         resource scope

  Timeout                   Slow downstream call,        Logs, VPC routes,
                            network issue, short timeout security groups,
                                                         timeout

  Schedule does not run     Disabled schedule, wrong     Scheduler status,
                            time zone, bad target/role   target, role, logs

  API returns error         Wrong                        API Gateway logs,
                            route/integration/response   status, headers, body
                            format                       

  Duplicate actions         Retries or duplicate         Idempotency and guard
                            delivery                     conditions

  Cannot reach database     VPC, routes, DNS, security   Subnets, routes,
                            groups                       endpoints, security
                                                         groups
  ------------------------------------------------------------------------------

Never log passwords, access keys, tokens, or sensitive personal data.

## 16. Security and production best practices

1.  Follow least privilege for execution roles.
2.  Never embed AWS access keys in code; use the execution role.
3.  Store secrets in Secrets Manager or Parameter Store as appropriate.
4.  Validate incoming events.
5.  Add idempotency for actions that create resources or change
    production state.
6.  Choose timeout, memory, and concurrency intentionally.
7.  Configure retries and failure handling.
8.  Use versions and aliases for controlled releases and rollbacks.
9.  Monitor errors, duration, throttling, and dependency failures.
10. Protect public APIs with authorization and throttling.
11. Do not assume a Lambda function attached to a VPC automatically has
    internet access.
12. Clean up lab resources and review costs.

## 17. Interview questions and answers

**Q1. What is AWS Lambda?** A serverless compute service that runs code
in response to events without requiring you to manage underlying
servers.

**Q2. What is a handler?** The entry point Lambda invokes,
e.g. `lambda_handler(event, context)`.

**Q3. Event vs. context?** Event contains input; context contains
runtime metadata such as request ID and remaining time.

**Q4. What is a cold start?** Extra initialization that can occur when
Lambda prepares a new execution environment.

**Q5. Is warm reuse guaranteed?** No. Lambda can create or remove
execution environments at any time.

**Q6. What is Provisioned Concurrency?** It keeps a configured number of
environments initialized to reduce startup latency, with additional
cost.

**Q7. What is an execution role?** An IAM role Lambda assumes so code
can call permitted AWS APIs.

**Q8. Synchronous vs. asynchronous?** Synchronous callers wait for a
response; asynchronous callers hand off an event without waiting for the
final result.

**Q9. What is concurrency?** Number of invocations running
simultaneously.

**Q10. How do you schedule Lambda?** Use EventBridge Scheduler with a
role that can invoke the intended function.

**Q11. Can Lambda run for hours?** No; one invocation has a maximum
timeout of 15 minutes.

**Q12. Is Lambda highly available?** The service is managed and highly
available, but application availability also depends on code,
configuration, dependencies, quotas, and architecture.

**Q13. Why idempotency?** Retries or duplicate delivery can process an
event more than once; idempotency prevents unintended duplicate effects.

**Q14. Can Lambda manage EC2?** Yes, if its execution role allows the
EC2 API actions and the Region/network configuration is correct.

**Q15. Does a successful Lambda test publish a website?** No. HTTP
access needs a front end such as API Gateway, configured
routing/integration, a valid response, and access controls.

## 18. One-line revision map

-   **Lambda:** Run code without managing underlying servers.
-   **Event:** Input describing what happened or was requested.
-   **Trigger:** Mechanism that invokes the function.
-   **Handler:** Function entry point.
-   **Cold start:** New environment needs initialization.
-   **Warm start:** Existing environment is reused.
-   **Execution role:** Permissions for code to access AWS services.
-   **Invocation:** Request to run a function.
-   **Concurrency:** Number of simultaneous executions.
-   **EventBridge Scheduler:** Runs Lambda on a schedule.
-   **CloudWatch:** Logs, metrics, and alarms.
-   **Idempotency:** Repeating an event does not create an unintended
    duplicate effect.

## 19. Classroom cleanup checklist

-   [ ] Delete the EventBridge schedule when no longer needed.
-   [ ] Stop or terminate test EC2 instances.
-   [ ] Check for unattached EBS volumes, Elastic IPs, snapshots, and
    other billable resources.
-   [ ] Remove temporary permissions that are no longer required.
-   [ ] Ensure no secrets were pasted into code or logs.
-   [ ] Review CloudWatch log retention.

**Trainer's note:** Keep dashboard numbers static for the first demo. In
a follow-up, read real metrics from a protected source. For EC2
automation, test with one explicitly tagged non-production instance
before enabling a schedule.
