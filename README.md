# Hard Target

A serverless solution to provide security assessments of newly built AMIs.

This project relies heavily on Amazon Web Services (AWS).

## Logic

When a new AMI is built, an event is published onto EventBridge.
This project captures that event and initiates a set of processes.
The first process is to spin up an EC2 instance using the newly built
AMI.  Once the server has started, Lynis is installed and then run so
that it generates a report.  This should take about 3 minutes.  Once the
report is ready, it is uploaded to an S3 bucket for permanent storage.
Finally, the server shuts itself down.

When the report lands in the S3 bucket, an event notification occurs
which notifies Event Bridge.  This invokes two Lambda functions.  The
first shuts down the EC2 instance.  The second inspects the report, pulls
out the score, then tags the AMI with the score.

And finally, there is a Lambda function which runs every 10 minutes to
check whether there are any old EC2 instances still sitting around doing
nothing.  Perhaps they got stuck.  It terminates them.

### Sequence Diagrams

#### Assessing an AMI

```mermaid
sequenceDiagram
  title Assessing an AMI

  participant AMI
  participant EB as EventBridge
  participant L as Lambda
  participant CFN as CloudFormation
  participant EC2
  participant S3

  AMI->>EB: publish event
  EB->>L: invokes
  L->>CFN: launches
  CFN->>L: << success >>
  CFN->>EC2: creates
  EC2->>EC2: run user data
  EC2->>S3: uploads report
  S3->>EC2: << success >>
  S3->>EB: puts event
```

#### Cleaning Up

```mermaid
sequenceDiagram
  title Cleaning Up

  participant EB as EventBridge
  participant L as Lambda
  participant CFN as CloudFormation
  participant EC2
  participant S3

  EB->>L: invokes
  L->>CFN: deletes
  CFN->>EC2: terminates
  EC2-->>CFN: << success >>
  CFN->>L: << success >>
```

#### Inspecting the Report

```mermaid
sequenceDiagram
  title Inspecting the Report

  participant AMI
  participant EB as EventBridge
  participant L as Lambda
  participant CFN as CloudFormation
  participant EC2
  participant S3
  participant SNS

  EB->>L: invokes
  L->>S3: get report
  S3-->>L: << file >>
  L->>L: get score from report
  L->>L: get AMI from event
  L->>AMI: tags
```

#### Reaping Stuck EC2 Instances

```mermaid
sequenceDiagram
  title Reaping Stuck EC2 Instances

  participant EB as EventBridge
  participant L as Lambda
  participant CFN as CloudFormation

  EB->>EB: timer fires every 10 mins
  EB->>L: invokes
  L->>CFN: list old hardtarget stacks
  CFN-->>L: << list(stacks) >>
  loop "each stack"
    L->>CFN: delete stack
  end
```

## Goals

This project lives by the following goals:

1. Do not interfere with normal servers.  This would break immutable infrastructure environments.
1. Do not impact the existing infrastructure environment.  Play well with others.
1. Do not cost too much money.  Security is still seen as a cost centre.
1. Do not keep persistent servers around.
1. Gather data, do not apply policy.  AMIs are tagged with a score.  It is up to service teams to implement their own standards of what is secure.

