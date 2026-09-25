# CorpWeb — CloudFormation Stack

SEIS 616 homework: `corpweb.json` is an AWS CloudFormation template that builds a VPC with two public subnets and two Amazon Linux 2023 web servers behind an Application Load Balancer. This README describes the template and reports how the stack was launched, verified, and deleted, including the command outputs.

## Architecture

```
Internet
  │  HTTP :80
  ▼
EngineeringLB  (Application Load Balancer, internet-facing, SG: WebserversSG)
  │  listener :80 HTTP → target group EngineeringWebservers (:80, health check HTTP :80 "/")
  ├─────────────────────────────────┬─────────────────────────────────┐
  ▼                                 ▼                                 │
web1  PublicSubnet1 10.0.0.0/24     web2  PublicSubnet2 10.0.1.0/24   │
      (AZ 1)                              (AZ 2)                      │
  └──────────────── EngineeringVpc 10.0.0.0/18 ───────────────────────┘
        route table: 0.0.0.0/0 → Internet Gateway
        WebserversSG: 22 ← YourIp, 80 ← 0.0.0.0/0
```

## Parameters

| Parameter      | Type                         | Notes                                               |
| -------------- | ---------------------------- | --------------------------------------------------- |
| `InstanceType` | String                       | `t2.micro` (default) or `t2.small` only             |
| `KeyPair`      | `AWS::EC2::KeyPair::KeyName` | Existing EC2 key pair for SSH                       |
| `YourIp`       | String (CIDR pattern)        | Your workstation's public IP, e.g. `203.0.113.7/32` |

**Output:** `WebUrl` gives the DNS name of `EngineeringLB`.

## Requirements → resources

| Requirement                                                                                                                                       | Logical resource(s)                                                                                                                                                                        |
| ------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| VPC `10.0.0.0/18`                                                                                                                                 | `EngineeringVpc`                                                                                                                                                                           |
| Subnets `10.0.0.0/24` and `10.0.1.0/24`, routed to the Internet                                                                                   | `PublicSubnet1`, `PublicSubnet2` (two different AZs), `InternetGateway`, `VpcGatewayAttachment`, `PublicRouteTable`, `PublicRoute` (`0.0.0.0/0 → IGW`), two `SubnetRouteTableAssociation`s |
| Two Amazon Linux 2023 instances, one per subnet, with `Name` tags, instance type from `InstanceType`, key from `KeyPair`, and the given user data | `web1`, `web2`                                                                                                                                                                             |
| Security group `WebserversSG` (22 from `YourIp`, 80 from anywhere)                                                                                | `WebserversSG` (`GroupName: WebserversSG`)                                                                                                                                                 |
| ALB `EngineeringLB`, target group `EngineeringWebservers`, 80 → 80 over HTTP, health check HTTP :80 `/`                                           | `EngineeringLB`, `EngineeringWebservers`, `EngineeringLBListener`                                                                                                                          |
| Output `WebUrl`                                                                                                                                   | `Outputs.WebUrl` = `EngineeringLB.DNSName`                                                                                                                                                 |

### Design notes

- **AMI:** `ImageId` uses the dynamic reference `{{resolve:ssm:/aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64}}`. The stack always gets the current Amazon Linux 2023 AMI, no AMI ID is hardcoded, and the template keeps exactly the three required parameters.
- **IAM role for the user data:** The required user data runs `aws s3 cp s3://seis665-public/index.php ...`. On an instance with no credentials the AWS CLI fails with "Unable to locate credentials", even though the object is public. `WebServerRole` and `WebServerInstanceProfile` give the instances `s3:GetObject` on `arn:aws:s3:::seis665-public/*` only, so the user data runs exactly as given. **Launching therefore requires `CAPABILITY_IAM`.** In the console, tick "I acknowledge that AWS CloudFormation might create IAM resources".
- **Instance metadata:** `index.php` reads the instance ID with a plain IMDSv1 request. The AL2023 AMI is flagged `ImdsSupport: v2.0`, so its instances require IMDSv2 tokens by default and the page would print a blank ID. `WebServerLaunchTemplate` sets `HttpTokens: optional` so the page shows which instance served each request.

## Launching

```bash
aws cloudformation create-stack --region us-east-1 --stack-name WebserversDev \
  --template-body file://corpweb.json --capabilities CAPABILITY_IAM \
  --parameters ParameterKey=InstanceType,ParameterValue=t2.micro \
               ParameterKey=KeyPair,ParameterValue=<your-key-pair> \
               ParameterKey=YourIp,ParameterValue=$(curl -s https://checkip.amazonaws.com)/32
aws cloudformation wait stack-create-complete --region us-east-1 --stack-name WebserversDev
aws cloudformation describe-stacks --region us-east-1 --stack-name WebserversDev \
  --query "Stacks[0].Outputs[?OutputKey=='WebUrl'].OutputValue" --output text
```

---

# Verification report

**Environment:** personal AWS account `762760349846`, region `us-east-1`, stack `WebserversDev`, run on 2026-09-25 (UTC). Verification used the **AWS CLI** (each step below), **SSH** into both instances, `curl`, and a **browser** check of the `WebUrl`.

## 1. Template validation

```
$ aws cloudformation validate-template --profile aaron@gesm4267 --region us-east-1 --template-body file://corpweb.json --query '{Parameters:Parameters[].ParameterKey,Capabilities:Capabilities,Reason:CapabilitiesReason}' --output json
{
    "Parameters": [
        "KeyPair",
        "YourIp",
        "InstanceType"
    ],
    "Capabilities": [
        "CAPABILITY_IAM"
    ],
    "Reason": "The following resource(s) require capabilities: [AWS::IAM::Role]"
}
```

## 2. Deployed from the committed GitHub copy

The assignment warns that editors can silently change a template before it's committed. The stack was therefore launched from the file **downloaded from this GitHub repo**, after confirming it's byte-identical to the local file. (In the commands below, local file paths are shortened. The actual run also passed `--profile aaron@gesm4267`.)

```
$ curl -s https://raw.githubusercontent.com/bucky-badger-gesmer/seis616-corpweb/main/corpweb.json -o corpweb.github.json
$ cmp corpweb.github.json corpweb.json && echo IDENTICAL
IDENTICAL
$ shasum -a 256 corpweb.github.json
f45631bd13e204270ad97aabc88d1028f625d1c0d7c7bf8c6e5fefb9a5e745f1  corpweb.github.json

$ aws cloudformation create-stack --region us-east-1 --stack-name WebserversDev \
    --template-body file://corpweb.github.json --capabilities CAPABILITY_IAM \
    --parameters ParameterKey=InstanceType,ParameterValue=t2.micro ParameterKey=KeyPair,ParameterValue=lab-key \
                 ParameterKey=YourIp,ParameterValue=x.x.x.x/32 --query StackId --output text
arn:aws:cloudformation:us-east-1:762760349846:stack/WebserversDev/bdc71d60-b919-11f1-aa7d-0affeb0dfddb
$ aws cloudformation wait stack-create-complete --region us-east-1 --stack-name WebserversDev
(exit 0, no failed events)
```

## 3. Stack status, parameters, and `WebUrl` output

```
$ aws sts get-caller-identity --profile aaron@gesm4267 --region us-east-1 --query '[Account,Arn]' --output text
762760349846	arn:aws:iam::762760349846:user/aaron
```

```
$ aws cloudformation describe-stacks --profile aaron@gesm4267 --region us-east-1 --stack-name WebserversDev --query 'Stacks[0].{Status:StackStatus,Created:CreationTime,Parameters:Parameters,Outputs:Outputs,Capabilities:Capabilities}' --output json
{
    "Status": "CREATE_COMPLETE",
    "Created": "2026-09-25T19:46:03.457000+00:00",
    "Parameters": [
        {
            "ParameterKey": "KeyPair",
            "ParameterValue": "lab-key"
        },
        {
            "ParameterKey": "YourIp",
            "ParameterValue": "x.x.x.x/32"
        },
        {
            "ParameterKey": "InstanceType",
            "ParameterValue": "t2.micro"
        }
    ],
    "Outputs": [
        {
            "OutputKey": "WebUrl",
            "OutputValue": "EngineeringLB-1369476053.us-east-1.elb.amazonaws.com",
            "Description": "DNS name of the EngineeringLB load balancer"
        }
    ],
    "Capabilities": [
        "CAPABILITY_IAM"
    ]
}
```

## 4. All logical resources created

```
$ aws cloudformation list-stack-resources --profile aaron@gesm4267 --region us-east-1 --stack-name WebserversDev --query 'StackResourceSummaries[].[LogicalResourceId,ResourceType,ResourceStatus,PhysicalResourceId]' --output table
----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
|                                                                                                    ListStackResources                                                                                                    |
+------------------------------------+--------------------------------------------+------------------+---------------------------------------------------------------------------------------------------------------------+
|  EngineeringLB                     |  AWS::ElasticLoadBalancingV2::LoadBalancer |  CREATE_COMPLETE |  arn:aws:elasticloadbalancing:us-east-1:762760349846:loadbalancer/app/EngineeringLB/aa94b21d4d6eff99                |
|  EngineeringLBListener             |  AWS::ElasticLoadBalancingV2::Listener     |  CREATE_COMPLETE |  arn:aws:elasticloadbalancing:us-east-1:762760349846:listener/app/EngineeringLB/aa94b21d4d6eff99/51bd4c7d8bebd056   |
|  EngineeringVpc                    |  AWS::EC2::VPC                             |  CREATE_COMPLETE |  vpc-0c549166008e1b24c                                                                                              |
|  EngineeringWebservers             |  AWS::ElasticLoadBalancingV2::TargetGroup  |  CREATE_COMPLETE |  arn:aws:elasticloadbalancing:us-east-1:762760349846:targetgroup/EngineeringWebservers/54d9de0e9c3b6c76             |
|  InternetGateway                   |  AWS::EC2::InternetGateway                 |  CREATE_COMPLETE |  igw-0514be4c741be0be3                                                                                              |
|  PublicRoute                       |  AWS::EC2::Route                           |  CREATE_COMPLETE |  rtb-036b234ff38dee310|0.0.0.0/0                                                                                    |
|  PublicRouteTable                  |  AWS::EC2::RouteTable                      |  CREATE_COMPLETE |  rtb-036b234ff38dee310                                                                                              |
|  PublicSubnet1                     |  AWS::EC2::Subnet                          |  CREATE_COMPLETE |  subnet-034171f3d538fe8d9                                                                                           |
|  PublicSubnet1RouteTableAssociation|  AWS::EC2::SubnetRouteTableAssociation     |  CREATE_COMPLETE |  rtbassoc-0e698a14ee48a6584                                                                                         |
|  PublicSubnet2                     |  AWS::EC2::Subnet                          |  CREATE_COMPLETE |  subnet-0fd56fbd357763fe5                                                                                           |
|  PublicSubnet2RouteTableAssociation|  AWS::EC2::SubnetRouteTableAssociation     |  CREATE_COMPLETE |  rtbassoc-00c8272fef15019fd                                                                                         |
|  VpcGatewayAttachment              |  AWS::EC2::VPCGatewayAttachment            |  CREATE_COMPLETE |  IGW|vpc-0c549166008e1b24c                                                                                          |
|  WebServerInstanceProfile          |  AWS::IAM::InstanceProfile                 |  CREATE_COMPLETE |  WebserversDev-WebServerInstanceProfile-Aawj1CWxV6ML                                                                |
|  WebServerLaunchTemplate           |  AWS::EC2::LaunchTemplate                  |  CREATE_COMPLETE |  lt-042d9d72b0cb235dd                                                                                               |
|  WebServerRole                     |  AWS::IAM::Role                            |  CREATE_COMPLETE |  WebserversDev-WebServerRole-CDZXPU8kBQb8                                                                           |
|  WebserversSG                      |  AWS::EC2::SecurityGroup                   |  CREATE_COMPLETE |  sg-0b780a161fed6f4af                                                                                               |
|  web1                              |  AWS::EC2::Instance                        |  CREATE_COMPLETE |  i-0ae4ac42201d170fc                                                                                                |
|  web2                              |  AWS::EC2::Instance                        |  CREATE_COMPLETE |  i-06563224368a9fe7b                                                                                                |
+------------------------------------+--------------------------------------------+------------------+---------------------------------------------------------------------------------------------------------------------+
```

## 5. Network: VPC, subnets, and Internet route

```
$ aws ec2 describe-vpcs --profile aaron@gesm4267 --region us-east-1 --vpc-ids vpc-0c549166008e1b24c --query 'Vpcs[].{VpcId:VpcId,Cidr:CidrBlock,Name:Tags[?Key==`aws:cloudformation:logical-id`]|[0].Value}' --output table
------------------------------------------------------------
|                       DescribeVpcs                       |
+-------------+------------------+-------------------------+
|    Cidr     |      Name        |          VpcId          |
+-------------+------------------+-------------------------+
|  10.0.0.0/18|  EngineeringVpc  |  vpc-0c549166008e1b24c  |
+-------------+------------------+-------------------------+
```

```
$ aws ec2 describe-subnets --profile aaron@gesm4267 --region us-east-1 --filters Name=vpc-id,Values=vpc-0c549166008e1b24c --query 'Subnets[].{LogicalId:Tags[?Key==`aws:cloudformation:logical-id`]|[0].Value,SubnetId:SubnetId,Cidr:CidrBlock,AZ:AvailabilityZone,PublicIpOnLaunch:MapPublicIpOnLaunch}' --output table
------------------------------------------------------------------------------------------------
|                                        DescribeSubnets                                       |
+------------+--------------+----------------+-------------------+-----------------------------+
|     AZ     |    Cidr      |   LogicalId    | PublicIpOnLaunch  |          SubnetId           |
+------------+--------------+----------------+-------------------+-----------------------------+
|  us-east-1b|  10.0.1.0/24 |  PublicSubnet2 |  True             |  subnet-0fd56fbd357763fe5   |
|  us-east-1a|  10.0.0.0/24 |  PublicSubnet1 |  True             |  subnet-034171f3d538fe8d9   |
+------------+--------------+----------------+-------------------+-----------------------------+
```

```
$ aws ec2 describe-route-tables --profile aaron@gesm4267 --region us-east-1 --filters Name=vpc-id,Values=vpc-0c549166008e1b24c Name=tag:aws:cloudformation:logical-id,Values=PublicRouteTable --query 'RouteTables[].{Routes:Routes[].[DestinationCidrBlock,GatewayId,State],Subnets:Associations[].SubnetId}' --output json
[
    {
        "Routes": [
            [
                "10.0.0.0/18",
                "local",
                "active"
            ],
            [
                "0.0.0.0/0",
                "igw-0514be4c741be0be3",
                "active"
            ]
        ],
        "Subnets": [
            "subnet-0fd56fbd357763fe5",
            "subnet-034171f3d538fe8d9"
        ]
    }
]
```

## 6. Instances: web1 and web2

Names, subnets, AZs, instance type, key pair, AMI, instance profile, and state:

```
$ aws ec2 describe-instances --profile aaron@gesm4267 --region us-east-1 --filters Name=vpc-id,Values=vpc-0c549166008e1b24c --query 'Reservations[].Instances[].{Name:Tags[?Key==`Name`]|[0].Value,InstanceId:InstanceId,Type:InstanceType,State:State.Name,Subnet:SubnetId,AZ:Placement.AvailabilityZone,ImageId:ImageId,KeyName:KeyName,PublicIp:PublicIpAddress,IMDSv2:MetadataOptions.HttpTokens,Profile:IamInstanceProfile.Arn}' --output table
----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
|                                                                                                                          DescribeInstances                                                                                                                         |
+------------+-----------+------------------------+----------------------+----------+-------+--------------------------------------------------------------------------------------------------+-----------------+----------+---------------------------+------------+
|     AZ     |  IMDSv2   |        ImageId         |     InstanceId       | KeyName  | Name  |                                             Profile                                              |    PublicIp     |  State   |          Subnet           |   Type     |
+------------+-----------+------------------------+----------------------+----------+-------+--------------------------------------------------------------------------------------------------+-----------------+----------+---------------------------+------------+
|  us-east-1b|  optional |  ami-0fef201115eefe936 |  i-06563224368a9fe7b |  lab-key |  web2 |  arn:aws:iam::762760349846:instance-profile/WebserversDev-WebServerInstanceProfile-Aawj1CWxV6ML  |  44.211.147.153 |  running |  subnet-0fd56fbd357763fe5 |  t2.micro  |
|  us-east-1a|  optional |  ami-0fef201115eefe936 |  i-0ae4ac42201d170fc |  lab-key |  web1 |  arn:aws:iam::762760349846:instance-profile/WebserversDev-WebServerInstanceProfile-Aawj1CWxV6ML  |  3.235.150.34   |  running |  subnet-034171f3d538fe8d9 |  t2.micro  |
+------------+-----------+------------------------+----------------------+----------+-------+--------------------------------------------------------------------------------------------------+-----------------+----------+---------------------------+------------+
```

```
$ aws ec2 describe-images --profile aaron@gesm4267 --region us-east-1 --image-ids ami-0fef201115eefe936 --query 'Images[].[ImageId,Name,Architecture]' --output text
ami-0fef201115eefe936	al2023-ami-2023.12.20260918.0-kernel-6.18-x86_64	x86_64
```

## 7. Security group

```
$ aws ec2 describe-security-groups --profile aaron@gesm4267 --region us-east-1 --filters Name=vpc-id,Values=vpc-0c549166008e1b24c Name=group-name,Values=WebserversSG --query 'SecurityGroups[].{GroupName:GroupName,GroupId:GroupId,Ingress:IpPermissions[].{Port:FromPort,Proto:IpProtocol,From:IpRanges[].CidrIp}}' --output json
[
    {
        "GroupName": "WebserversSG",
        "GroupId": "sg-0b780a161fed6f4af",
        "Ingress": [
            {
                "Port": 80,
                "Proto": "tcp",
                "From": [
                    "0.0.0.0/0"
                ]
            },
            {
                "Port": 22,
                "Proto": "tcp",
                "From": [
                    "x.x.x.x/32"
                ]
            }
        ]
    }
]
```

## 8. Load balancer, listener, target group, and health

```
$ aws elbv2 describe-load-balancers --profile aaron@gesm4267 --region us-east-1 --names EngineeringLB --query 'LoadBalancers[].{Name:LoadBalancerName,DNS:DNSName,Type:Type,Scheme:Scheme,State:State.Code,AZs:AvailabilityZones[].ZoneName,SGs:SecurityGroups}' --output json
[
    {
        "Name": "EngineeringLB",
        "DNS": "EngineeringLB-1369476053.us-east-1.elb.amazonaws.com",
        "Type": "application",
        "Scheme": "internet-facing",
        "State": "active",
        "AZs": [
            "us-east-1a",
            "us-east-1b"
        ],
        "SGs": [
            "sg-0b780a161fed6f4af"
        ]
    }
]
```

```
$ aws elbv2 describe-listeners --profile aaron@gesm4267 --region us-east-1 --load-balancer-arn arn:aws:elasticloadbalancing:us-east-1:762760349846:loadbalancer/app/EngineeringLB/aa94b21d4d6eff99 --query 'Listeners[].{Port:Port,Protocol:Protocol,Action:DefaultActions[0].Type,TargetGroup:DefaultActions[0].TargetGroupArn}' --output json
[
    {
        "Port": 80,
        "Protocol": "HTTP",
        "Action": "forward",
        "TargetGroup": "arn:aws:elasticloadbalancing:us-east-1:762760349846:targetgroup/EngineeringWebservers/54d9de0e9c3b6c76"
    }
]
```

```
$ aws elbv2 describe-target-groups --profile aaron@gesm4267 --region us-east-1 --names EngineeringWebservers --query 'TargetGroups[].{Name:TargetGroupName,Protocol:Protocol,Port:Port,TargetType:TargetType,HC_Protocol:HealthCheckProtocol,HC_Port:HealthCheckPort,HC_Path:HealthCheckPath}' --output json
[
    {
        "Name": "EngineeringWebservers",
        "Protocol": "HTTP",
        "Port": 80,
        "TargetType": "instance",
        "HC_Protocol": "HTTP",
        "HC_Port": "80",
        "HC_Path": "/"
    }
]
```

```
$ aws elbv2 describe-target-health --profile aaron@gesm4267 --region us-east-1 --target-group-arn arn:aws:elasticloadbalancing:us-east-1:762760349846:targetgroup/EngineeringWebservers/54d9de0e9c3b6c76 --query 'TargetHealthDescriptions[].[Target.Id,Target.Port,TargetHealth.State]' --output table
------------------------------------------
|          DescribeTargetHealth          |
+----------------------+-----+-----------+
|  i-06563224368a9fe7b |  80 |  healthy  |
|  i-0ae4ac42201d170fc |  80 |  healthy  |
+----------------------+-----+-----------+
```

## 9. Requests are distributed across both instances

Each response prints the ID of the instance that served it. Both IDs appear, and 20 requests split 11 / 9.

```
$ curl -s -i -m 5 http://EngineeringLB-1369476053.us-east-1.elb.amazonaws.com/
HTTP/1.1 200 OK
Date: Fri, 25 Sep 2026 19:50:00 GMT
Content-Type: text/html; charset=UTF-8
Transfer-Encoding: chunked
Connection: keep-alive
Server: Apache/2.4.68 (Amazon Linux)
X-Powered-By: PHP/8.5.10

Hi, I'm instance i-06563224368a9fe7b
```

```
$ for i in $(seq 10); do curl -s http://EngineeringLB-1369476053.us-east-1.elb.amazonaws.com/index.php; done
Hi, I'm instance i-0ae4ac42201d170fc
Hi, I'm instance i-06563224368a9fe7b
Hi, I'm instance i-0ae4ac42201d170fc
Hi, I'm instance i-06563224368a9fe7b
Hi, I'm instance i-0ae4ac42201d170fc
Hi, I'm instance i-06563224368a9fe7b
Hi, I'm instance i-0ae4ac42201d170fc
Hi, I'm instance i-06563224368a9fe7b
Hi, I'm instance i-0ae4ac42201d170fc
Hi, I'm instance i-0ae4ac42201d170fc
```

```
$ ... | sort | uniq -c   (distribution over 20 requests)
  11 Hi, I'm instance i-06563224368a9fe7b
   9 Hi, I'm instance i-0ae4ac42201d170fc
```

**Browser:** opening `http://EngineeringLB-1369476053.us-east-1.elb.amazonaws.com` in a web browser loaded the `Hi, I'm instance …` page from the stack.

## 10. SSH into the instances

SSH used the stack's `KeyPair` (`lab-key`) from the `YourIp` workstation. On each instance: Apache is active, the user data copied `index.php` from S3 (so the IAM role worked), and the local page reports the instance's own ID.

```
$ ssh -i ~/.ssh/lab-key.pem -o IdentitiesOnly=yes -o StrictHostKeyChecking=accept-new -o ConnectTimeout=10 -o BatchMode=yes ec2-user@3.235.150.34 'echo "connected as $(whoami) on $(hostname)"; cat /etc/os-release | grep PRETTY_NAME; echo -n "instance-id: "; TOKEN=$(curl -s -X PUT http://169.254.169.254/latest/api/token -H "X-aws-ec2-metadata-token-ttl-seconds: 60"); curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/instance-id; echo; echo -n "httpd: "; systemctl is-active httpd; ls -l /var/www/html/; curl -s http://localhost/index.php'
Warning: Permanently added '3.235.150.34' (ED25519) to the list of known hosts.
connected as ec2-user on ip-10-0-0-93.ec2.internal
PRETTY_NAME="Amazon Linux 2023.12.20260918"
instance-id: i-0ae4ac42201d170fc
httpd: active
total 4
-rw-r--r--. 1 root root 142 Oct 14  2018 index.php
Hi, I'm instance i-0ae4ac42201d170fc
```

```
$ ssh -i ~/.ssh/lab-key.pem -o IdentitiesOnly=yes -o StrictHostKeyChecking=accept-new -o ConnectTimeout=10 -o BatchMode=yes ec2-user@44.211.147.153 'echo "connected as $(whoami) on $(hostname)"; cat /etc/os-release | grep PRETTY_NAME; echo -n "instance-id: "; TOKEN=$(curl -s -X PUT http://169.254.169.254/latest/api/token -H "X-aws-ec2-metadata-token-ttl-seconds: 60"); curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/instance-id; echo; echo -n "httpd: "; systemctl is-active httpd; ls -l /var/www/html/; curl -s http://localhost/index.php'
Warning: Permanently added '44.211.147.153' (ED25519) to the list of known hosts.
connected as ec2-user on ip-10-0-1-106.ec2.internal
PRETTY_NAME="Amazon Linux 2023.12.20260918"
instance-id: i-06563224368a9fe7b
httpd: active
total 4
-rw-r--r--. 1 root root 142 Oct 14  2018 index.php
Hi, I'm instance i-06563224368a9fe7b
```

## 11. Teardown

The stack was deleted with CloudFormation, and every resource was confirmed gone. Each lookup returns "not found". The `WebserversSG` lookup returns an empty list, and the last check shows the `WebUrl` no longer resolves (curl exit 6).

```
$ aws sts get-caller-identity --profile aaron@gesm4267 --region us-east-1 --query '[Account,Arn]' --output text
762760349846	arn:aws:iam::762760349846:user/aaron
```

```
$ aws cloudformation delete-stack --profile aaron@gesm4267 --region us-east-1 --stack-name WebserversDev
```

```
$ aws cloudformation wait stack-delete-complete --profile aaron@gesm4267 --region us-east-1 --stack-name WebserversDev && echo stack-delete-complete: OK
stack-delete-complete: OK
```

```
$ aws cloudformation describe-stacks --profile aaron@gesm4267 --region us-east-1 --stack-name WebserversDev
aws: [ERROR]: An error occurred (ValidationError) when calling the DescribeStacks operation: Stack with id WebserversDev does not exist
```

```
$ aws cloudformation list-stacks --profile aaron@gesm4267 --region us-east-1 --stack-status-filter DELETE_COMPLETE --query 'StackSummaries[?StackName==`WebserversDev`]|[0].[StackName,StackStatus,DeletionTime]' --output text
WebserversDev	DELETE_COMPLETE	2026-09-25T19:52:24.981000+00:00
```

```
$ aws ec2 describe-instances --profile aaron@gesm4267 --region us-east-1 --instance-ids i-0ae4ac42201d170fc i-06563224368a9fe7b --query 'Reservations[].Instances[].[Tags[?Key==`Name`]|[0].Value,InstanceId,State.Name]' --output text
web2	i-06563224368a9fe7b	terminated
web1	i-0ae4ac42201d170fc	terminated
```

```
$ aws elbv2 describe-load-balancers --profile aaron@gesm4267 --region us-east-1 --names EngineeringLB
aws: [ERROR]: An error occurred (LoadBalancerNotFound) when calling the DescribeLoadBalancers operation: Load balancers '[EngineeringLB]' not found
```

```
$ aws elbv2 describe-target-groups --profile aaron@gesm4267 --region us-east-1 --names EngineeringWebservers
aws: [ERROR]: An error occurred (TargetGroupNotFound) when calling the DescribeTargetGroups operation: One or more target groups not found
```

```
$ aws ec2 describe-vpcs --profile aaron@gesm4267 --region us-east-1 --vpc-ids vpc-0c549166008e1b24c
aws: [ERROR]: An error occurred (InvalidVpcID.NotFound) when calling the DescribeVpcs operation: The vpc ID 'vpc-0c549166008e1b24c' does not exist
```

```
$ aws ec2 describe-security-groups --profile aaron@gesm4267 --region us-east-1 --filters Name=group-name,Values=WebserversSG --query 'SecurityGroups[].GroupId' --output text
```

```
$ aws iam get-role --profile aaron@gesm4267 --region us-east-1 --role-name WebserversDev-WebServerRole-CDZXPU8kBQb8
aws: [ERROR]: An error occurred (NoSuchEntity) when calling the GetRole operation: The role with name WebserversDev-WebServerRole-CDZXPU8kBQb8 cannot be found.
```

```
$ curl -s -m 5 -o /dev/null -w 'HTTP %{http_code} (curl exit follows)\n' http://EngineeringLB-1369476053.us-east-1.elb.amazonaws.com/ ; echo curl exit: $?
HTTP 000 (curl exit follows)
curl exit: 6
```
