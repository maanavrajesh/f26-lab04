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

```
$ aws cloudformation create-stack --stack-name lab04-service     --template-body file://infra/template.yaml     --parameters file://infra/params-scenario2.json
{
    "StackId": "arn:aws:cloudformation:us-east-1:293021197474:stack/lab04-service/470da830-b37c-11f1-973b-0eca24e9c8f9"
}
$ aws cloudformation wait stack-create-complete --stack-name lab04-service
$ aws cloudformation describe-stacks --stack-name lab04-service     --query "Stacks[0].Outputs[].[OutputKey,OutputValue]" --output table
-------------------------------------------------------------------------
|                            DescribeStacks                             |
+------------+----------------------------------------------------------+
|  InstanceId|  i-017d5739cfc721eee                                     |
|  ServiceUrl|  http://ec2-52-90-222-246.compute-1.amazonaws.com:8080   |
+------------+----------------------------------------------------------+
```

### Healthy redeploy after the scenario 2 fix (Milestone 2)

```
$ aws cloudformation delete-stack --stack-name lab04-service
$ aws cloudformation wait stack-delete-complete --stack-name lab04-service
$ aws cloudformation create-stack --stack-name lab04-service     --template-body file://infra/template.yaml     --parameters file://infra/params-healthy.json
{
    "StackId": "arn:aws:cloudformation:us-east-1:293021197474:stack/lab04-service/fc4c66c0-b37f-11f1-9c3d-0affd46b34d1"
}
$ aws cloudformation wait stack-create-complete --stack-name lab04-service
$ aws cloudformation describe-stacks --stack-name lab04-service     --query "Stacks[0].Outputs[].[OutputKey,OutputValue]" --output table
------------------------------------------------------------------------
|                            DescribeStacks                            |
+------------+---------------------------------------------------------+
|  InstanceId|  i-04a4331e0d2d25852                                    |
|  ServiceUrl|  http://ec2-184-73-21-51.compute-1.amazonaws.com:8080   |
+------------+---------------------------------------------------------+
```

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

Run from my laptop against the scenario 2 URL. The first two are 1 and 2 minutes after
`CREATE_COMPLETE`; the third is ~25 minutes later, after the diagnosis below, so this
is not the normal warm-up window:

```
$ curl http://ec2-52-90-222-246.compute-1.amazonaws.com:8080/api/health
curl: (7) Failed to connect to ec2-52-90-222-246.compute-1.amazonaws.com port 8080 after 2216 ms: Could not connect to server

$ curl http://ec2-52-90-222-246.compute-1.amazonaws.com:8080/api/health
curl: (7) Failed to connect to ec2-52-90-222-246.compute-1.amazonaws.com port 8080 after 2207 ms: Could not connect to server

$ curl http://ec2-52-90-222-246.compute-1.amazonaws.com:8080/api/health
curl: (7) Failed to connect to ec2-52-90-222-246.compute-1.amazonaws.com port 8080 after 2176 ms: Could not connect to server
```

**The log line that told you what was wrong:**

Shell on the instance through Systems Manager (`aws ssm start-session --target
i-017d5739cfc721eee`; the transcript below was captured by sending the same commands
through SSM Run Command, `aws ssm send-command --document-name AWS-RunShellScript`,
because I was driving it from a non-interactive tool):

```
sh-5.2$ sudo docker ps
CONTAINER ID   IMAGE                                     COMMAND                  CREATED          STATUS          PORTS                                       NAMES
6d1db2491910   ghcr.io/cmu-17-214/lab04-service:latest   "/__cacert_entrypoin…"   23 minutes ago   Up 23 minutes   0.0.0.0:8080->8080/tcp, :::8080->8080/tcp   lab04-service

sh-5.2$ sudo docker logs lab04-service
lab04-service listening on 9090
```

Supporting evidence from the same session:

```
sh-5.2$ sudo docker exec lab04-service env | grep PORT
PORT=9090

sh-5.2$ sudo ss -ltnp | grep -E ':8080|:9090'
LISTEN 0      4096         0.0.0.0:8080       0.0.0.0:*    users:(("docker-proxy",pid=19197,fd=5))
LISTEN 0      4096            [::]:8080          [::]:*    users:(("docker-proxy",pid=19252,fd=5))

sh-5.2$ sudo grep -E 'EFFECTIVE_PORT=|docker run' /var/log/cloud-init-output.log
+ EFFECTIVE_PORT=9090
+ docker run -d --name lab04-service --restart unless-stopped -p 8080:8080 -e PORT=9090 ghcr.io/cmu-17-214/lab04-service:latest
```

**What was wrong, and the fix you applied:**

The service inside the container was listening on port **9090**, but everything
outside the container was wired for **8080**. `params-scenario2.json` sets
`PortOverride` to `"9090"`; in `infra/template.yaml` the UserData script copies that
into `EFFECTIVE_PORT` (line 96) and passes it as `-e PORT=9090` (line 107), which is
the port the app binds, and `docker logs` confirms it: `lab04-service listening on
9090`. The host-to-container mapping on line 106 is `-p ${ServicePort}:${ServicePort}`,
which is still `8080:8080`, and the security group (lines 48-49) and `ServiceUrl`
output (line 114) also still say 8080. So the `docker ps` line and the `docker logs`
line contradict each other: Docker's proxy accepts connections on host 8080 and
forwards them to container port 8080, where nothing is listening, and the
connection gets reset, which curl reports as "Could not connect". `docker ps` says
"Up" because the container is fine; it is just listening on the wrong side of the
port mapping. The fix was the infrastructure one: I did not touch the running
container. I deleted the broken stack and created it again with
`infra/params-healthy.json`, where `PortOverride` is empty so `EFFECTIVE_PORT` falls
back to `ServicePort` (lines 97-99) and the container gets `-e PORT=8080`, matching
the mapping, the security group, and the URL.

**The healthy curl after the fix:**

From my laptop, ~50 seconds after the recreated stack hit `CREATE_COMPLETE`:

```
$ curl http://ec2-184-73-21-51.compute-1.amazonaws.com:8080/api/health
{"status":"ok"}
```

And on the new instance the two lines that disagreed before now agree:

```
sh-5.2$ sudo docker ps
ghcr.io/cmu-17-214/lab04-service:latest  Up 24 seconds  0.0.0.0:8080->8080/tcp, :::8080->8080/tcp  lab04-service
sh-5.2$ sudo docker logs lab04-service
lab04-service listening on 8080
sh-5.2$ sudo docker exec lab04-service env | grep PORT
PORT=8080
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
