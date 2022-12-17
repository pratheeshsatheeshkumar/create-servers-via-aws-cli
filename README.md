# Implementation of multiple  public and private servers in a VPC  using AWS-CLI.

This document is intended to devops engineers those who want to build AWS infrastructure solely using AWS-CLI. I will try to explain each step as simple as I can keeping the beginners in mind. Later, I will make another document for automating these activities using terraform.

- Goal of this mini project is to create three servers in  a VPC.
- frontend, bastion and backend are the proposed servers.
- bation and frontend have to be in a public subnet.
- backend has to be in a Private subnet which should not be accessible from external world.
- Bastion has access from public internet via ssh
- frontend is accessible via SSH from  bastion and via HTTP from internet.
- backend is able to access the internet via NAT Gateway.
- AWS console is not supposed to use for any activity.

![Alt text](https://github.com/pratheeshsatheeshkumar/create-servers-via-aws-cli/blob/main/zomato.png "Block diagram of the expected architecture.")

#### 1. AWS CLI configuration. 
_As a prerequisite an IAM user with programmatic access has to be available in hand._
```sh
$ aws configure
AWS Access Key ID [None]:XXXXXXXXXXXX
AWS Secret Access Key [None]: XXXXXXXXXXXXXXXX
Default region name [None]: ap-south-1
Default output format [None]: json
```
#### 2. Creation of a Virtual Private Cloud for the project.

_The following command creates a VPC named “zomato” in a CIDR block of 172.16.0.0/16.
It will retrieve the ID value of the VPC and save it to the zomato_vpc_id file for future use. 
Name tag is also attached with the VPC._
```sh
$ zomato_vpc_id=$(aws ec2 create-vpc --cidr-block 172.16.0.0/16 --tag-specifications 'ResourceType=vpc, Tags=[{Key=Name,Value=zomato}]' --query Vpc.VpcId --output text)

echo $zomato_vpc_id > zomato_vpc_id

echo $zomato_vpc_id

vpc-0a9a5b7b11beccb6d
```
_Do not forget to enable DNS hostnames._
```sh
aws ec2 modify-vpc-attribute --vpc-id "$(<zomato_vpc_id)" --enable-dns-hostnames "{\"Value\":true}"
``` 
![Snapshot of zomato VPC.](https://screenrec.com/share/xIp3yUQf8G)
#### 3. To display the zomato VPC by providing its ID from the file.
```sh
$ aws ec2 describe-vpcs --vpc-ids=$(<zomato_vpc_id)
{
    "Vpcs": [
        {
            "CidrBlock": "172.16.0.0/16",
            "DhcpOptionsId": "dopt-0695867080c66d8a3",
            "State": "available",
            "VpcId": "vpc-0cb1d8fadd9d93258",
            "OwnerId": "566209955933",
            "InstanceTenancy": "default",
            "CidrBlockAssociationSet": [
                {
                    "AssociationId": "vpc-cidr-assoc-0c9c2ff52540c39d7",
                    "CidrBlock": "172.16.0.0/16",
                    "CidrBlockState": {
                        "State": "associated"
                    }
                }
            ],
            "IsDefault": false,
            "Tags": [
                {
                    "Key": "Name",
                    "Value": "zomato"
                }
            ]
        }
    ]
}
```

#### 4. Creation of  public and private subnets is the next step. 
_4a. You have to create  a public subnet called zomato_public_1 in  ap-south-1a AZ with a 172.16.0.0/18 CIDR block._ _You have to place zomato-frontend server in this subnet later._
```sh
$ zomato_public_1=$(aws ec2 create-subnet --vpc-id $zomato_vpc_id --cidr-block 172.16.0.0/18 --availability-zone ap-south-1a   --query Subnet.SubnetId --output text)

$ echo $ zomato_public_1> zomato_public_1

$ echo $zomato_public_1

subnet-0ee1b6c035da5bd1b

#Addition of a Name tag is an essential requirement.

$ aws ec2 create-tags --resources $zomato_public_1 --tags Key=Name,Value=zomato_public_1

```
_You can view the subnet using the describe-subnets command._
```sh
 aws ec2 describe-subnets --subnet-id $(<zomato_public_1)

{
    "Subnets": [
        {
            "AvailabilityZone": "ap-south-1a",
            "AvailabilityZoneId": "aps1-az1",
            "AvailableIpAddressCount": 16379,
            "CidrBlock": "172.16.0.0/18",
            "DefaultForAz": false,
            "MapPublicIpOnLaunch": false,
            "MapCustomerOwnedIpOnLaunch": false,
            "State": "available",
            "SubnetId": "subnet-02bf2e40861c3ba65",
            "VpcId": "vpc-0cb1d8fadd9d93258",
            "OwnerId": "566209955933",
            "AssignIpv6AddressOnCreation": false,
            "Ipv6CidrBlockAssociationSet": [],
            "SubnetArn": "arn:aws:ec2:ap-south-1:566209955933:subnet/subnet-02bf2e40861c3ba65",
            "EnableDns64": false,
            "Ipv6Native": false,
            "PrivateDnsNameOptionsOnLaunch": {
                "HostnameType": "ip-name",
                "EnableResourceNameDnsARecord": false,
                "EnableResourceNameDnsAAAARecord": false
            }
        }
    ]
}

```
_4b. Secondly, You have to create  another public subnet called zomato_public_2 in  ap-south-1b AZ with a 172.16.64.0/18 CIDR block.__You have to place zomato-bastion server in this subnet later._

```sh
$ zomato_public_2=$(aws ec2 create-subnet --vpc-id $zomato_vpc_id --cidr-block 172.16.64.0/18 --availability-zone ap-south-1a --query Subnet.SubnetId --output text)

$ echo $ zomato_public_2> zomato_public_2

$ echo $zomato_public_2

subnet-06c1206444325f124

#Addition of a Name tag is an essential requirement.

$ aws ec2 create-tags --resources $zomato_public_1 --tags Key=Name,Value=zomato_public_2
```
_You can view the subnet using the describe-subnets command in JSON format_
```sh
aws ec2 describe-subnets --subnet-id $(<zomato_public_2)
{
    "Subnets": [
        {
            "AvailabilityZone": "ap-south-1a",
            "AvailabilityZoneId": "aps1-az1",
            "AvailableIpAddressCount": 16379,
            "CidrBlock": "172.16.64.0/18",
            "DefaultForAz": false,
            "MapPublicIpOnLaunch": false,
            "MapCustomerOwnedIpOnLaunch": false,
            "State": "available",
            "SubnetId": "subnet-0e5197656e8bd5924",
            "VpcId": "vpc-0cb1d8fadd9d93258",
            "OwnerId": "566209955933",
            "AssignIpv6AddressOnCreation": false,
            "Ipv6CidrBlockAssociationSet": [],
            "SubnetArn": "arn:aws:ec2:ap-south-1:566209955933:subnet/subnet-0e5197656e8bd5924",
            "EnableDns64": false,
            "Ipv6Native": false,
            "PrivateDnsNameOptionsOnLaunch": {
                "HostnameType": "ip-name",
                "EnableResourceNameDnsARecord": false,
                "EnableResourceNameDnsAAAARecord": false
            }
        }
    ]
}
```
_4c. Finally, It is time to create a private subnet named zomato_private_1 in  ap-south-1b AZ with a 172.16.128.0/18 CIDR block._ _You have to install zomato-backend server in this subnet later._
```sh
$ zomato_private_1=$(aws ec2 create-subnet --vpc-id $(<zomato_vpc_id) --cidr-block 172.16.128.0/18 --availability-zone ap-south-1a  --query Subnet.SubnetId --output text)

$ echo $ zomato_private_1> zomato_private-1

$ echo $zomato_private_1

subnet-0a520cdda86437631

#Addition of a Name tag.

$ aws ec2 create-tags --resources $zomato_private_1 --tags Key=Name,Value=zomato_private_1
```
_You can view the subnet using the describe-subnets command in JSON format_
```sh
$ aws ec2 describe-subnets --subnet-id $(<zomato_public_1)
{
    "Subnets": [
        {
            "AvailabilityZone": "ap-south-1a",
            "AvailabilityZoneId": "aps1-az1",
            "AvailableIpAddressCount": 16379,
            "CidrBlock": "172.16.0.0/18",
            "DefaultForAz": false,
            "MapPublicIpOnLaunch": false,
            "MapCustomerOwnedIpOnLaunch": false,
            "State": "available",
            "SubnetId": "subnet-02bf2e40861c3ba65",
            "VpcId": "vpc-0cb1d8fadd9d93258",
            "OwnerId": "566209955933",
            "AssignIpv6AddressOnCreation": false,
            "Ipv6CidrBlockAssociationSet": [],
            "SubnetArn": "arn:aws:ec2:ap-south-1:566209955933:subnet/subnet-02bf2e40861c3ba65",
            "EnableDns64": false,
            "Ipv6Native": false,
            "PrivateDnsNameOptionsOnLaunch": {
                "HostnameType": "ip-name",
                "EnableResourceNameDnsARecord": false,
                "EnableResourceNameDnsAAAARecord": false
            }
        }
    ]
}
```
_4d. It is important to assign public IP addresses to the public subnets to gain external access._
```sh
$ aws ec2 modify-subnet-attribute --subnet-id $(<zomato_public_1) --map-public-ip-on-launch

$ aws ec2 modify-subnet-attribute --subnet-id $(<zomato_public_2) --map-public-ip-on-launch
```
#### 5. You have to create an Internet Gateway and attach it with zomato VPC.
```sh
$ IGW=$(aws ec2 create-internet-gateway  --query InternetGateway.InternetGatewayId --output text)

$ echo $IGW>IGW

$ echo $(<IGW)
igw-02255bc7ed0b958ea

$ aws ec2 attach-internet-gateway --internet-gateway-id $(<IGW) --vpc-id $zomato_vpc_id
```
_Now , Internet gateway is created and attached with zomato VPC._
#### 6. NAT Gateway creation and Elastic IP allocation for it.
_a. For a NAT-GW to access external devices it is requied to assign an Elastic IP._
```sh
$ elastic_id=$(aws ec2 allocate-address --query AllocationId --output text)

$ echo $elastic_id > elastic_id

$ echo $(<elastic_id)
eipalloc-0a3834ba4d2f2c5b0
```
_b.NAT GW creation is an important step for the zomato_backend server in the private subnet to access external nodes._
```sh
$ NGW=$(aws ec2 create-nat-gateway --subnet-id $(<zomato_public_2) --allocation-id $(<elastic_id) --query NatGateway.NatGatewayId --output text)

$ echo $NGW
nat-01b189b169b8dadf6
$ echo $NGW>NGW
```

#### 7. Update the default route table for public subnets  and create a custom one for private subnets.

_a.A default route table is created automatically with the creation of zomato VPC._ _You have to get the id of this route table to modify routes and add tag it as default-route-table._
```sh
$ default_rt=$(aws ec2 describe-route-tables  --filters "Name=vpc-id,Values=$zomato_vpc_id" --query 'RouteTables[].Associations[].RouteTableId' --output text)

$ echo $default_rt>default_rt

$ echo $default_rt

rtb-01417bdcf47d7ab38
```
_Creation of Name Tag is necessary for future automations._
```sh
$ aws ec2 create-tags --resources $default_rt --tags Key=Name,Value=default-route-table
```
_Associating  default route table with zomato_public_1 subnet._
```sh
aws ec2 associate-route-table  --subnet-id $(<zomato_public_1) --route-table-id $(<default_rt)
{
    "AssociationId": "rtbassoc-073c67d17dbd39cb7",
    "AssociationState": {
        "State": "associated"
    }
}
```
_Associating  default route table with zomato_public_2 subnet._
```sh
$ aws ec2 associate-route-table  --subnet-id "subnet-0e5197656e8bd5924"  --route-table-id $(<default_rt)
{
    "AssociationId": "rtbassoc-0dd0fa50d80af2f21",
    "AssociationState": {
        "State": "associated"
    }
}

```
_b. You have to create a custom route table for the private subnet : zomato_private_1 and associate it with the private subnet._
 ```sh
$ custom_rt=$(aws ec2 create-route-table --vpc-id $(<zomato_vpc_id) --query RouteTable.RouteTableId --output text)

$ echo $custom_rt>custom_rt

$ echo $(<custom_rt)
rtb-0cab76a99b1743af7

#Associating custome route table with the zomato_private_1 subnet

$  aws ec2 associate-route-table  --subnet-id $(<zomato_private_1) --route-table-id $(<custom_rt)
{
    "AssociationId": "rtbassoc-0c510f382799aa930",
    "AssociationState": {
        "State": "associated"
    }
}
``` 
_c. You have to add a route table entry for external access through Internet Gateway._
```sh
$ aws ec2 create-route --route-table-id $(<custom_rt) --destination-cidr-block 0.0.0.0/0 --gateway-id $(<IGW)
{
    "Return": true
}
```

_d.Add route table entry in the _custom route table_ to enable instances in the private subnet to communicate to the external world through the NAT gateway._
```sh
$ aws ec2 create-route --route-table-id $(<custom_rt) --destination-cidr-block 0.0.0.0/0 --nat-gateway-id $(<NGW)
{
    "Return": true
}
```
_You can confirm the successful creation of the route in the corresponding route table from the following command._
````sh
$ aws ec2 describe-route-tables --filters "Name=vpc-id,Values=$(<zomato_vpc_id)"
{
    "RouteTables": [
        {
            "Associations": [
                {
                    "Main": false,
                    "RouteTableAssociationId": "rtbassoc-0c510f382799aa930",
                    "RouteTableId": "rtb-0cab76a99b1743af7",
                    "SubnetId": "subnet-0a520cdda86437631",
                    "AssociationState": {
                        "State": "associated"
                    }
                }
            ],
            "PropagatingVgws": [],
            "RouteTableId": "rtb-0cab76a99b1743af7",
            "Routes": [
                {
                    "DestinationCidrBlock": "172.16.0.0/16",
                    "GatewayId": "local",
                    "Origin": "CreateRouteTable",
                    "State": "active"
                },
                {
                    "DestinationCidrBlock": "0.0.0.0/0",
                    "NatGatewayId": "nat-00e084ecdaac578f2",
                    "Origin": "CreateRoute",
                    "State": "active"
                }
            ],
            "Tags": [],
            "VpcId": "vpc-0cb1d8fadd9d93258",
            "OwnerId": "566209955933"
        },
        {
            "Associations": [
                {
                    "Main": true,
                    "RouteTableAssociationId": "rtbassoc-074b42f513b977e05",
                    "RouteTableId": "rtb-0a39ab8c771a5ae9d",
                    "AssociationState": {
                        "State": "associated"
                    }
                },
                {
                    "Main": false,
                    "RouteTableAssociationId": "rtbassoc-073c67d17dbd39cb7",
                    "RouteTableId": "rtb-0a39ab8c771a5ae9d",
                    "SubnetId": "subnet-02bf2e40861c3ba65",
                    "AssociationState": {
                        "State": "associated"
                    }
                },
                {
                    "Main": false,
                    "RouteTableAssociationId": "rtbassoc-0dd0fa50d80af2f21",
                    "RouteTableId": "rtb-0a39ab8c771a5ae9d",
                    "SubnetId": "subnet-0e5197656e8bd5924",
                    "AssociationState": {
                        "State": "associated"
                    }
                }
            ],
            "PropagatingVgws": [],
            "RouteTableId": "rtb-0a39ab8c771a5ae9d",
            "Routes": [
                {
                    "DestinationCidrBlock": "172.16.0.0/16",
                    "GatewayId": "local",
                    "Origin": "CreateRouteTable",
                    "State": "active"
                }
            ],
            "Tags": [
                {
                    "Key": "Name",
                    "Value": "default-route-table"
                }
            ],
            "VpcId": "vpc-0cb1d8fadd9d93258",
            "OwnerId": "566209955933"
        }
    ]
}
````

#### 8. Creation of Security groups for the servers.
_a. Initially, you have to create a security group for the zomato-bastion server_
```sh
$ zomato_bastion_sg=$(aws ec2 create-security-group --group-name zomato_bastion_sg --description "Allow SSH access" --vpc-id $(<zomato_vpc_id) --output text)

$ echo $zomato_bastion_sg
sg-0b5ef2570a6cb4e97

$ echo $zomato_bastion_sg>zomato_bastion_sg
```
_b. Secondly, you have to create a security group for the zomato-frontend server._
```sh
$ zomato_frontend_sg=$(aws ec2 create-security-group --group-name zomato_frontend_sg --description "Allow SSH access from bastion and  http access from all" --vpc-id $(<zomato_vpc_id) --output text)

$ echo $zomato_frontend_sg>zomato_frontend_sg

$ echo $zomato_frontend_sg
sg-007729d89205b2cb4
```
_c. Finally, it is time to create a security group for the zomato-backend server._
```sh
$ zomato_backend_sg=$(aws ec2 create-security-group --group-name zomato_backend_sg --description "Allow SSH access from bastion and  http access from all" --vpc-id $(<zomato_vpc_id) --output text)

$ echo $zomato_backend_sg>zomato_backend_sg

$ echo $zomato_backend_sg
sg-007729d8920ds5b2dsd
```
#### 9. Configure the inbound rules of these security groups.
_a. Add a rule to allow SSH access to the bastion server._
```sh
$ aws ec2 authorize-security-group-ingress --group-id $(<zomato_bastion_sg) --protocol tcp --port 22 --cidr 0.0.0.0/0
{
    "Return": true,
    "SecurityGroupRules": [
        {
            "SecurityGroupRuleId": "sgr-0b617ef9af3090c42",
            "GroupId": "sg-0b5ef2570a6cb4e97",
            "GroupOwnerId": "566209955933",
            "IsEgress": false,
            "IpProtocol": "tcp",
            "FromPort": 22,
            "ToPort": 22,
            "CidrIpv4": "0.0.0.0/0"
        }
    ]
}

```
_b. Add a rule to allow SSH access to the zomato.frontend server from zomato.bastion server._
```sh
$ aws ec2 authorize-security-group-ingress --group-id $(<zomato_frontend_sg) --protocol tcp --port 22 --source-group $(<zomato_bastion_sg)
{
    "Return": true,
    "SecurityGroupRules": [
        {
            "SecurityGroupRuleId": "sgr-000b0d084ef0392b3",
            "GroupId": "sg-007729d89205b2cb4",
            "GroupOwnerId": "566209955933",
            "IsEgress": false,
            "IpProtocol": "tcp",
            "FromPort": 22,
            "ToPort": 22,
            "ReferencedGroupInfo": {
                "GroupId": "sg-0b5ef2570a6cb4e97",
                "UserId": "566209955933"
            }
        }
    ]
}
```
_c. Add a rule to allow HTTP access to the zomato.frontend server from the internet._
```sh
$ aws ec2 authorize-security-group-ingress --group-id $(<zomato_frontend_sg) --protocol tcp --port 80 --cidr 0.0.0.0/0
{
    "Return": true,
    "SecurityGroupRules": [
        {
            "SecurityGroupRuleId": "sgr-053c5cd48fa4b72ae",
            "GroupId": "sg-007729d89205b2cb4",
            "GroupOwnerId": "566209955933",
            "IsEgress": false,
            "IpProtocol": "tcp",
            "FromPort": 80,
            "ToPort": 80,
            "CidrIpv4": "0.0.0.0/0"
        }
    ]
}
```
_d. Add a rule to allow SSH access to the zomato.backend server from the zomato.bastion server._
```sh
$ aws ec2 authorize-security-group-ingress --group-id $(<zomato_backend_sg) --protocol tcp --port 22 --source-group $(<zomato_bastion_sg)
{
    "Return": true,
    "SecurityGroupRules": [
        {
            "SecurityGroupRuleId": "sgr-01e3a2ba9a220a325",
            "GroupId": "sg-026e068136ed361c9",
            "GroupOwnerId": "566209955933",
            "IsEgress": false,
            "IpProtocol": "tcp",
            "FromPort": 22,
            "ToPort": 22,
            "ReferencedGroupInfo": {
                "GroupId": "sg-0b5ef2570a6cb4e97",
                "UserId": "566209955933"
            }
        }
    ]
}
```
_e. Add a rule to enable the zomato.frontend server to access mysql on the zomato.backend server._
```sh
$ aws ec2 authorize-security-group-ingress --group-id $(<zomato_backend_sg) --protocol tcp --port 3306 --source-group $(<zomato_bastion_sg)
{
    "Return": true,
    "SecurityGroupRules": [
        {
            "SecurityGroupRuleId": "sgr-0ce07bc6a49e266b0",
            "GroupId": "sg-026e068136ed361c9",
            "GroupOwnerId": "566209955933",
            "IsEgress": false,
            "IpProtocol": "tcp",
            "FromPort": 3306,
            "ToPort": 3306,
            "ReferencedGroupInfo": {
                "GroupId": "sg-0b5ef2570a6cb4e97",
                "UserId": "566209955933"
            }
        }
    ]
}
```
#### 10. You have to create a key pair to connect securely  with the servers.
```sh
$ aws ec2 create-key-pair --key-name zomato.pem --query 'KeyMaterial' --output text > zomato.pem  

$ chmod 400 zomato.pem
```
#### 11 . Finding out the desired AMI ID for the Instance creation.
_I have used Amazon Linux 2 Kernel 5.10as the AMI for instance creation._
```sh
$ ami_id=$(aws ec2 describe-images --owners self amazon  --filter "Name=description,Values=Amazon Linux 2 Kernel 5.10 AMI*" --query Images[0].ImageId --output text)

$ echo $ami_id>ami_id

$ echo $(<ami_id)
ami-0d03faa951a539717
```
#### 12 . Now it is time to launch your servers.
_a.Launch the zomato.bastion server with the required security group, keypair, subnet, and AMI._
```sh
bastion_id=$(aws ec2 run-instances --image-id $(<ami_id) --count 1 --instance-type t2.micro --key-name zomato.pem --security-group-ids $(<zomato_bastion_sg) --subnet-id $(<zomato_public_2) --query Instances[].InstanceId --output text)

$ echo $bastion_id
i-04f79e932e4b61106

$ echo $bastion_id>bastion_id
```
_b.Launch the zomato.frontend server with the required security group, keypair, subnet, and AMI._
```sh
frontend_id=$(aws ec2 run-instances --image-id $(<ami_id) --count 1 --instance-type t2.micro --key-name zomato.pem --security-group-ids $(<zomato_frontend_sg) --subnet-id $(<zomato_public_1) --query Instances[].InstanceId --output text)

$ echo $frontend_id>frontend_id

$ echo $(<frontend_id)
i-074859d35f8a0a563
```
_c.Launch the zomato.backend server with the required security group, keypair, subnet, and AMI._
```sh
$ backend_id=$(aws ec2 run-instances --image-id $(<ami_id) --count 1 --instance-type t2.micro --key-name zomato.pem --security-group-ids $(<zomato_backend_sg) --subnet-id $(<zomato_private_1) --query Instances[].InstanceId --output text)

$ echo $backend_id>backend_id

$ echo $(<backend_id)
i-03afa76d05c474ba1
```
_d. Let us add Name tags to the instances to identify them easily._
```sh
aws ec2 create-tags --resources $(<bastion_id) --tags Key=Name,Value=zomato.bastion

aws ec2 create-tags --resources $(<frontend_id) --tags Key=Name,Value=zomato.frontend

aws ec2 create-tags --resources $(<backend_id) --tags Key=Name,Value=zomato.backend
```
![Alt text](https://github.com/pratheeshsatheeshkumar/create-servers-via-aws-cli/blob/main/zomato%20EC2%20instances.png "Created Instances."


#### 13 . Connecting to zomato.bastion server via ssh from my workstation using public ip and hostname.
```sh
$ sudo ssh -i zomato.pem ec2-user@43.205.144.202
The authenticity of host '43.205.144.202 (43.205.144.202)' can't be established.
ECDSA key fingerprint is SHA256:0/bVdHUGcsZwOeLaK/3WWPkudzWKXT7tWuQigATuYFU.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '43.205.144.202' (ECDSA) to the list of known hosts.

       __|  __|_  )
       _|  (     /   Amazon Linux 2 AMI
      ___|\___|___|

https://aws.amazon.com/amazon-linux-2/
[ec2-user@ip-172-16-57-179 ~]$
```
#### 14 . Connecting to zomato.frontend from zomato.bastion server via ssh using private ip and hostname.
```sh
[ec2-user@ip-172-16-57-179 ~]$ sudo ssh -i zomato.pem  ec2-user@172.16.19.0
The authenticity of host '172.16.19.0 (172.16.19.0)' can't be established.
ECDSA key fingerprint is SHA256:9kZPCM0bbTiN5NhEScYtUc84STx7U/1eLqgN/s2QKhg.
ECDSA key fingerprint is MD5:8f:f6:3c:52:60:ad:04:d9:7c:0c:05:f8:f4:ce:64:00.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '172.16.19.0' (ECDSA) to the list of known hosts.

       __|  __|_  )
       _|  (     /   Amazon Linux 2 AMI
      ___|\___|___|

https://aws.amazon.com/amazon-linux-2/
[ec2-user@ip-172-16-19-0 ~]$
```
#### 15. Connecting to zomato.backend from zomato.bastion server via ssh using private ip and hostname.
```sh
[ec2-user@ip-172-16-57-179 ~]$ sudo ssh -i zomato.pem  ec2-user@172.16.173.178
The authenticity of host '172.16.173.178 (172.16.173.178)' can't be established.
ECDSA key fingerprint is SHA256:ciU0DTgFdQx0QGqFQhcFtcoG4vresoM10zKgeUN36AI.
ECDSA key fingerprint is MD5:5f:32:f4:af:dd:08:d4:11:a9:11:29:36:27:98:47:6f.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '172.16.173.178' (ECDSA) to the list of known hosts.

       __|  __|_  )
       _|  (     /   Amazon Linux 2 AMI
      ___|\___|___|

https://aws.amazon.com/amazon-linux-2/
[ec2-user@ip-172-16-173-178 ~]$
````
#### 16. zomato.backend is able to access external servers via NAT Gateway.
```sh
[ec2-user@ip-172-16-173-178 ~]$ sudo yum install mariadb-server -y
Loaded plugins: extras_suggestions, langpacks, priorities, update-motd
Resolving Dependencies
--> Running transaction check
---> Package mariadb-server.x86_64 1:5.5.68-1.amzn2 will be installed
--> Processing Dependency: mariadb(x86-64) = 1:5.5.68-1.amzn2 for package: 1:mariadb-server-5.5.68-1.amzn2.x86_64
--> Processing Dependency: perl-DBI for package: 1:mariadb-server-5.5.68-1.amzn2.x86_64
```
Now we can install the required packages to the servers and setup the required websites by keeping the zomato.backend in the private network safely. I tried my level best to explain each steps. Hope this document will help you to setup the AWS infrastructure easily. I am working on a bash script to automate these tasks. I will publish it once I complete the documentation. 
