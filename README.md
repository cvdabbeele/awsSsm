## Introduction
This repo describes how to setup AWS Systems Manager to automatically deploy the Trend Micro Cloud One Workload Security Agent (C1WSA) to any EC2 that has specific tags.   
This would happen at the creation time or later when the matching tag is added (in that case, the EC2 requires a reboot to load the AWS SSM agent)

## Prerequisites
- A default VPC, or a VPC with a public Subnet and a Internet Gateway attached
- An IAM user with an AWS Access Key and an AWS Secret Key
- 
## Setup a Cloud9 work environment
- Environment type: Create a new EC2 instance for environment (direct access)
- Instance type: t2.micro (1 GiB RAM + 1 vCPU)
- Platform: *Ubuntu Server 18.04 LTS*
- You must have a default VPC, or a VPC with a public Subnet and a Internet Gateway attached
   
### Register for Cloud One in Marketplace
login  
select Cloud One Workload Security  

### add AWS account to Cloud One Workload Security
To Do 


## Configure AWS Systems Manager
#see also: https://cloudone.trendmicro.com/docs/workload-security/aws-systems-manager/#protect-your-computers

### create Parameters for Trend Micro Cloud One Workload Security
```	AWS Services -> AWS Systems Manager (-> Get started with Systems Manager) -> Parameter store -> Create parameter
		NAME:				VALUE:
		dsActivationUrl 	dsm://agents.deepsecurity.trendmicro.com:443/
		dsManagerUrl 	    https://app.deepsecurity.trendmicro.com:443
		dsTenantId 	        <your_tenant_id>
		dsToken             <your_ds_token>
```
### create a Distributor
 		AWS -> Systems Manager -> Distributor -> Third Party -> TrendMicro-CloudOne-WorkloadSecurity -> Install on Schedule  
		name: (e.g.) DistributorForDsaForC1ws  
		Action: Install  
		Installation Type: In-place update  
		Name: TrendMicro-CloudOne-WorkloadSecurity  
		Targets: Specify instance tags  
			Tag key: c1ws 
			Tag value: enabled 
			"ADD" (dont forget to click the Add button !!)
		keep everything else default
		Create Association  
		In the next screen, Select the Association you just created -> click "View Details" -> Execution history (tab) -> Success (this can take a while)  

### create instanceProfile for SSM  

export AWS_INSTANCEPROFILE_FOR_SSM='instanceProfileForSSM'
export AWS_SERVICEROLE_FOR_SSM='ServiceRoleForSSM'  #this is an existing Role
#export AWS_SERVICEROLE_FOR_SSM='AmazonSSMRoleForInstancesQuickSetup'

cat  <<-EOF  >./SSMassumeRolePolicy.json
{
  "Version": "2012-10-17",
  "Statement": {
    "Effect": "Allow",
    "Principal": { "Service": "ec2.amazonaws.com" },
    "Action": "sts:AssumeRole"
  }
}
EOF
cat ./SSMassumeRolePolicy.json

aws iam create-role --role-name ${AWS_SERVICEROLE_FOR_SSM} --assume-role-policy-document file://./SSMassumeRolePolicy.json
aws iam attach-role-policy --role-name ${AWS_SERVICEROLE_FOR_SSM} --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore
aws iam attach-role-policy --role-name ${AWS_SERVICEROLE_FOR_SSM} --policy-arn arn:aws:iam::aws:policy/AmazonSSMPatchAssociation

aws iam create-instance-profile --instance-profile-name ${AWS_INSTANCEPROFILE_FOR_SSM}
aws iam add-role-to-instance-profile --instance-profile-name ${AWS_INSTANCEPROFILE_FOR_SSM} --role-name ${AWS_SERVICEROLE_FOR_SSM}




## Create and auto-protect new instances


### Create keypair
```
export AWS_KEYNAME="MacDefault"
aws ec2  create-key-pair --key-name $AWS_KEYNAME
```

### find AMI ID for AmazonLinux2 AMI
```
AWS_AMI_AZLINUX_ID=`aws ec2 describe-images \
--owners 'amazon' \
--filters 'Name=name,Values=amzn2-ami-hvm-2.0.*' 'Name=state,Values=available' \
--query 'sort_by(Images, &CreationDate)[-1].[ImageId]' \
--output 'text'`
#echo $AWS_AMI_AZLINUX_ID
#aws ec2 describe-images --image-ids $AWS_AMI_AZLINUX_ID      
#--filters 'Name=description,Values=Amazon Linux 2 AMI*' 'Name=state,Values=available' \
#export AWS_AMI_AZLINUX_ID='ami-0de9f803fcac87f46'
```

### find AMI ID for Ubuntu18 AMI
```
AWS_AMI_UBUNTU18_ID=`aws ec2 describe-images \
 --filters Name=name,Values=ubuntu/images/hvm-ssd/ubuntu-bionic-18.04-amd64-server* \
 --query 'Images[*].[ImageId]' --output text \
 | sort -k2 -r \
 | head -n1`
echo $AWS_AMI_UBUNTU18_ID
#aws ec2 describe-images --image-ids $AWS_AMI_UBUNTU18_ID      
```

### find AMI ID for Ubuntu20 AMI
```
AWS_AMI_UBUNTU20_ID=`aws ec2 describe-images \
 --filters Name=name,Values=ubuntu/images/hvm-ssd/ubuntu-focal-20.04-amd64-server* \
 --query 'Images[*].[ImageId]' --output text \
 | sort -k2 -r \
 | head -n1`
#echo $AWS_AMI_UBUNTU20_ID
#aws ec2 describe-images --image-ids $AWS_AMI_UBUNTU20_ID      
#Ubuntu Server 20.04 LTS (HVM), SSD Volume Type - ami-0767046d1677be5a0 (64-bit x86) / ami-08b6fc871ad49ff41 (64-bit Arm)
```

### TO DO: setup securitygroup to allow ssh from "My IP"

### create user-data file for ubuntu images
```
cat <<EOF >user-data.sh
#!/bin/bash
sudo snap install aws-cli --classic
EOF
```

## instantiate an Amazon Linux2 instance with the SSMRole Profile
```
AWS_EC2_AZLINUX_ID=`aws ec2 run-instances --image-id $AWS_AMI_AZLINUX_ID --count 1 --instance-type t2.micro --iam-instance-profile Name=${AWS_INSTANCEPROFILE_FOR_SSM} --key-name $AWS_KEYNAME --query 'Instances[0].InstanceId' --output text`
    # aws ec2 run-instances --image-id ami-xxxxxxxx --count 1 --instance-type t2.micro --key-name MyKeyPair --security-group-ids sg-903004f8 --subnet-id subnet-6e7f829e
echo $AWS_EC2_AZLINUX_ID
aws ec2 create-tags --resources $AWS_EC2_AZLINUX_ID --tags Key=Name,Value=AmazonLinux2Instance011 Key=c1ws,Value=enabled

## instantiate an Ubuntu18 instance with the SSMRole Profile
AWS_EC2_UBUNTU18_ID=`aws ec2 run-instances --image-id $AWS_AMI_UBUNTU18_ID --count 1 --instance-type t2.micro --user-data file://user-data.sh --iam-instance-profile Name=${AWS_INSTANCEPROFILE_FOR_SSM} --key-name $AWS_KEYNAME --query 'Instances[0].InstanceId' --output text`
     #  In order to use this AWS Marketplace product you need to accept terms and subscribe. To do so please visit https://aws.amazon.com/marketplace/pp?sku=a8jyynf4hjutohctm41o2z18m
echo $AWS_EC2_UBUNTU18_ID
aws ec2 create-tags --resources $AWS_EC2_UBUNTU18_ID --tags Key=Name,Value=Ubuntu18Instance011 Key=c1ws,Value=enabled
```

## instantiate an Ubuntu20 instance with the SSMRole Profile
```
#AWS_AMI_UBUNTU20_ID="ami-0767046d1677be5a0"

AWS_EC2_UBUNTU20_ID=`aws ec2 run-instances --image-id $AWS_AMI_UBUNTU20_ID --count 1 --instance-type t2.micro --user-data file://user-data.sh --iam-instance-profile Name=${AWS_INSTANCEPROFILE_FOR_SSM} --key-name $AWS_KEYNAME --query 'Instances[0].InstanceId' --output text`
     #  In order to use this AWS Marketplace product you need to accept terms and subscribe. To do so please visit https://aws.amazon.com/marketplace/pp?sku=a8jyynf4hjutohctm41o2z18m
echo $AWS_EC2_UBUNTU20_ID
aws ec2 create-tags --resources $AWS_EC2_UBUNTU20_ID --tags Key=Name,Value=Ubuntu20Instance011 Key=c1ws,Value=enabled
```



### Connect to the instance
To Do 
```    
ssh -i $AWS_KEYNAME ec2-user@ec2-18-223-180-43.us-east-2.compute.amazonaws.com
sudo systemctl restart amazon-ssm-agent
curl http://169.254.169.254
sudo tail -n 50 -f /var/log/amazon/ssm/amazon-ssm-agent.log
```

## Verify that the Instances are protected by Cloud One Workload Security
go to https://cloudone.trendmicro.com/home  
login
select Cloud One Workload Security



### to delete the role run the following commands

###aws ec2 disassociate-iam-instance-profile --association-id iip-assoc-?????
###aws iam describe-IamInstanceProfileAssociations ???

aws iam remove-role-from-instance-profile --instance-profile-name ${AWS_INSTANCEPROFILE_FOR_SSM} --role-name ${AWS_SERVICEROLE_FOR_SSM}  
aws iam detach-role-policy --role-name ${AWS_SERVICEROLE_FOR_SSM} --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore
aws iam detach-role-policy --role-name ${AWS_SERVICEROLE_FOR_SSM} --policy-arn arn:aws:iam::aws:policy/AmazonSSMPatchAssociation

#not needed; fails aws iam delete-role-policy --role-name ${AWS_SERVICEROLE_FOR_SSM}  --policy-name AmazonSSMManagedInstanceCore
#not needed; fails aws iam delete-role-policy --role-name ${AWS_SERVICEROLE_FOR_SSM}  --policy-name AmazonSSMPatchAssociation
aws iam delete-role --role-name ${AWS_SERVICEROLE_FOR_SSM}
aws iam delete-instance-profile --instance-profile-name ${AWS_INSTANCEPROFILE_FOR_SSM}
