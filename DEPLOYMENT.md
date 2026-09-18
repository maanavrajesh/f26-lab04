# Deployment Evidence

Fill this in as you go. Paste real output, not descriptions of output. A TA reads this
file with you at recitation.

## 1. Deployed URL and instance id

<!-- The ServiceUrl and InstanceId outputs. Paste both here every time
describe-stacks prints them, for the healthy deploy and for scenario 2. Both
change on every recreate, and you will need them for curls and sessions. -->

### Healthy deploy (Milestone 1)

```
$ aws cloudformation create-stack --stack-name lab04-service     --template-body file://infra/template.yaml     --parameters file://infra/params-healthy.json
{
    "StackId": "arn:aws:cloudformation:us-east-1:293021197474:stack/lab04-service/46169370-b37b-11f1-a866-0ef974d7294d"
}
$ aws cloudformation wait stack-create-complete --stack-name lab04-service
$ aws cloudformation describe-stacks --stack-name lab04-service     --query "Stacks[0].Outputs[].[OutputKey,OutputValue]" --output table
--------------------------------------------------------------------------
|                             DescribeStacks                             |
+------------+-----------------------------------------------------------+
|  InstanceId|  i-071125f4f21c147e9                                      |
|  ServiceUrl|  http://ec2-18-234-111-153.compute-1.amazonaws.com:8080   |
+------------+-----------------------------------------------------------+
```

Physical resources the stack reported:

```
$ aws cloudformation list-stack-resources --stack-name lab04-service     --query "StackResourceSummaries[].[LogicalResourceId,ResourceType,PhysicalResourceId,ResourceStatus]" --output table
+----------------------+--------------------------+---------------------------------------------------+-------------------+
|  ServiceInstance     |  AWS::EC2::Instance      |  i-071125f4f21c147e9                              |  CREATE_COMPLETE  |
|  ServiceSecurityGroup|  AWS::EC2::SecurityGroup |  lab04-service-ServiceSecurityGroup-wnkYTP7IR4LM  |  CREATE_COMPLETE  |
+----------------------+--------------------------+---------------------------------------------------+-------------------+
```

### Scenario 2 deploy (Milestone 2)

<!-- paste the scenario 2 outputs here -->

## 2. External health check

Run the check from your own machine, not from the instance. Paste the command and the
response.

Run from my laptop about 40 seconds after `CREATE_COMPLETE` (the first couple of
attempts got "couldn't connect" while the instance was still installing Docker and
pulling the image):

```
$ curl http://ec2-18-234-111-153.compute-1.amazonaws.com:8080/api/health
{"status":"ok"}

$ curl http://ec2-18-234-111-153.compute-1.amazonaws.com:8080/api/rooms
[
  {"id":"WEH-5202","name":"Wean 5202","capacity":40},
  {"id":"GHC-4401","name":"Gates 4401","capacity":24},
  {"id":"POS-146","name":"Posner 146","capacity":120},
  {"id":"TEP-2700","name":"Tepper 2700","capacity":16}
]
```

## 3. What the template created

Three or four sentences, your own words. What compute, what network access, and what
glue made the service start.

The stack created exactly two resources. **Compute:** one `t3.micro` EC2 instance
(`ServiceInstance`, `infra/template.yaml` lines 63-108) running the latest Amazon Linux
2023 AMI, which CloudFormation looked up at deploy time from the public SSM parameter
in the `AmiId` parameter (lines 32-35); it uses the pre-existing Learner Lab
`LabInstanceProfile` (line 72) so Session Manager can open a shell on it, and the
`vockey` key pair as an SSH fallback. **Network access:** one security group
(`ServiceSecurityGroup`, lines 42-61) in the default VPC that allows inbound TCP on
`ServicePort` (8080) and on 22 from `0.0.0.0/0`, with the default allow-all egress
left in place so the instance can reach the package repos and the container registry;
the instance is attached to it via `!GetAtt ServiceSecurityGroup.GroupId` (line 69),
which is also the implicit dependency that makes CloudFormation build the group
first. **Glue:** the instance's UserData bash script (lines 80-108) runs on first
boot: it `dnf install`s Docker, enables it with systemd, schedules a `shutdown -h +240`
safety timer, then runs
`docker run -d --name lab04-service --restart unless-stopped -p 8080:8080 -e PORT=8080 ghcr.io/cmu-17-214/lab04-service:latest`,
which pulls the public image and starts the service on the same 8080 that the
security group opens and that the `ServiceUrl` output (line 114) points at.

## 4. Scenario 2 diagnosis

**The failing curl** (command and output):

```

```

**The log line that told you what was wrong:**

```

```

**What was wrong, and the fix you applied:**

<!-- One or two sentences. Say what you changed and where you changed it. -->

**The healthy curl after the fix:**

```

```

## 5. Teardown proof

Paste the delete output, or describe the console evidence that the resources are gone.

### After Milestone 1 (deleted before redeploying for scenario 2)

```
$ aws cloudformation delete-stack --stack-name lab04-service
$ aws cloudformation wait stack-delete-complete --stack-name lab04-service
$ aws cloudformation describe-stacks --stack-name lab04-service
An error occurred (ValidationError) when calling the DescribeStacks operation: Stack with id lab04-service does not exist
$ aws ec2 describe-instances --instance-ids i-071125f4f21c147e9     --query "Reservations[].Instances[].State.Name" --output text
terminated
```

### Final teardown (Milestone 3)

```

```
