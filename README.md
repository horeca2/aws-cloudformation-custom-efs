# aws-cloudformation-restore-efs
Custom EFS with automatic recovery from backup

## Problem with current EFS implementation
When you fully restore an EFS snapshot to a new filesystem, the files are not placed in the root of the filesystem, as you might expect, but in a directory named aws-backup-restore_datetime.
This creates difficulties. You need to mount EFS via EC2 or Lambda, move files and delete that directory. Discussed here https://forums.aws.amazon.com/thread.jspa?threadID=346945
Additionally, AWS does not currently support restoring an EFS snapshot from CloudFormation. Although, for example, there is a special property for RDS called DBSnapshotIdentifier, if you specify it, the database will be restored automatically. It would be nice to have a similar property for EFS.
This solution proposes to completely eliminate the use of the AWS :: EFS :: FileSystem resource in your templates and replace it with a nested stack.

## Syntax
To declare this entity in the AWS CloudFormation template, use the following syntax:
YAML
``` yaml
Type: AWS::CloudFormation::Stack
  Properties:
    Parameters:
      BackupVaultName: String
      RecoveryPointArn: String
      Subnet: String
      SecurityGroup: String
      Vpc: String
      TemplateURL: String
      PerformanceMode: String
      Encrypted: Boolean
      KmsKeyId: String
      TransitionToIA: String
      TransitionToPrimaryStorageClass: String

```

> Unfortunately, at the moment the solution does not support all AWS :: EFS :: FileSystem properties which you can find here https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-efs-filesystem.html ... If you do not see a property, then it is not supported or is set to its default value and cannot be changed. Request properties in issues.

## Properties

### BackupVaultName
The name of the storage where the backup is stored

Required: Yes, if RecoveryPointArn is set

Type: String

Updating requires: Replacement

### RecoveryPointArn
Arn restore points. If this parameter is equal to an empty string, then an empty file system will be created.
To understand how this works and what the RecoveryPointArn property is for, see the description of the DBSnapshotIdentifier property in the RDS documentation. Behavior is similar
https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-properties-rds-database-instance.html#cfn-rds-dbinstance-dbsnapshotidentifier

Required: No

Type: String

Updating requires: Replacement

### Vpc
VPC ID, for example vpc-a123baa3. A mount target will be created in this VPC

Required: Yes

Type: String

Updating requires: Replacement

### Subnet
A mount target will be created on this subnet. The subnet must belong to the previously specified VPC

Required: Yes

Type: String

Updating requires: Replacement

### SecurityGroup
The security group used by the mount target. The security group must belong to the previously specified VPC

Required: Yes

Type: String

Updating requires: Replacement

### TemplateURL
The location of the file containing the body of the template. The URL must point to the template located in the Amazon S3 bucket. Copy the template.yaml file to your bucket and link to it here.

Required: Yes

Type: String

Update requires: ?

### PerformanceMode
The performance mode of the file system. We recommend generalPurpose performance mode for most file systems. File systems using the maxIO performance mode can scale to higher levels of aggregate throughput and operations per second with a tradeoff of slightly higher latencies for most file operations. The performance mode can't be changed after the file system has been created.

Required: No

Type: String

Allowed values: generalPurpose | maxIO

Update requires: Replacement

### Encrypted
A Boolean value that, if true, creates an encrypted file system. When creating an encrypted file system, you must specifying a KmsKeyId for an existing AWS KMS key. 

Required: No

Type: Boolean

Update requires: Replacement

### KmsKeyId
The ID of the AWS KMS key to be used to protect the encrypted file system. This parameter is required. This ID can be in one of the following formats:

* Key ID - A unique identifier of the key, for example 1234abcd-12ab-34cd-56ef-1234567890ab.

If KmsKeyId is specified, the Encrypted parameter must be set to true.

Required: Conditional

Type: String

Update requires: Replacement

### TransitionToIA
Describes the period of time that a file is not accessed, after which it transitions to IA storage. Metadata operations such as listing the contents of a directory don't count as file access events.

Required: No

Type: String

Valid Values: NONE | AFTER_7_DAYS | AFTER_14_DAYS | AFTER_30_DAYS | AFTER_60_DAYS | AFTER_90_DAYS

Update requires: No interruption

### TransitionToPrimaryStorageClass
Describes when to transition a file from IA storage to primary storage. Metadata operations such as listing the contents of a directory don't count as file access events.

Type: String

Valid Values: NONE | AFTER_1_ACCESS

Required: No

Update requires: No interruption

## Return values

### Fn :: GetAtt
An inner function returns the value for the specified attribute of this type. The available attributes and examples of return values ​​are shown below.

#### Outputs.Efs
Filesystem identifier, e.g. fs-06799d63c3aeb4707

## Examples

The following sample template mounts a filesystem to an EC2 instance. At any time, you can change the parameters of the BackupVaultName and RecoveryPointArn templates, and then CloudFormation will restore your file system from a backup, create a new EC2 instance and mount the restored file system to it. Then the old file system and the old instance will be safely removed. Great, isn't it? :)

``` YAML
AWSTemplateFormatVersion: 2010-09-09
Parameters: 
  InstanceType:
    Type: String
    Default: t2.micro
    AllowedValues:
      - t2.micro 
  ImageId:
    Description: AMI
    Type: AWS::EC2::Image::Id
    Default: ami-058e6df85cfc7760b
  KeyName:
    Description: Key pair. Must be created manualy
    Type: AWS::EC2::KeyPair::KeyName    
  Vpc:
    Type: AWS::EC2::VPC::Id    
  Subnet:
    Type: AWS::EC2::Subnet::Id   
  BackupVaultName:
    Type: String
    Default: Default
  RecoveryPointArn:
    Type: String
  TemplateURL:
    Type: String
    MinLength: 1   
    Description: URL for template.yaml. Should be in S3 
  PerformanceMode:
    Type: String
    AllowedValues:
      - generalPurpose
      - maxIO
    Default: generalPurpose  
  TransitionToIA:
    Type: String
    AllowedValues:
      - NONE
      - AFTER_14_DAYS
      - AFTER_30_DAYS
      - AFTER_60_DAYS
      - AFTER_7_DAYS
      - AFTER_90_DAYS
    Default: NONE   
  TransitionToPrimaryStorageClass:
    Type: String
    AllowedValues:
      - NONE
      - AFTER_1_ACCESS  
    Default: NONE   
  Encrypted:
    Type: String
    AllowedValues:
      - "true"
      - "false"
    Default: "false"  
  KmsKeyId:
    Type: String
Resources:
  LaunchTemplate:
    Type: AWS::EC2::LaunchTemplate
    Properties:
      LaunchTemplateData:       
        ImageId: !Ref ImageId       
        InstanceType: !Ref InstanceType     
        KeyName: !Ref KeyName       
        SecurityGroupIds:  
          - !Ref EC2SecurityGroup  
        UserData: !Base64 
          Fn::Sub: 
            - |
              #!/bin/bash
              yum update -y
              yum install -y amazon-efs-utils
              mkdir /home/ec2-user/efs 
              mount -t efs -o tls,accesspoint=${AccessPoint} ${EFS}:/ /home/ec2-user/efs            
            - EFS: !GetAtt Efs.Outputs.Efs   
  
  EC2SecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Allow SSH trafic      
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 22
          ToPort: 22
          CidrIp: 0.0.0.0/0        
      VpcId: !Ref Vpc
  
  EfsSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Allow trafic from EC2SecurityGroup
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 2049
          ToPort: 2049
          SourceSecurityGroupId: !Ref EC2SecurityGroup
      VpcId: !Ref Vpc
        
  Efs:
    Type: AWS::CloudFormation::Stack
    Properties:
      Parameters:
        BackupVaultName: !Ref BackupVaultName
        RecoveryPointArn: !Ref RecoveryPointArn
        Subnet: !Ref Subnet
        SecurityGroup: !Ref EfsSecurityGroup 
        Vpc: !Ref Vpc
        PerformanceMode: !Ref PerformanceMode
        TransitionToIA: !Ref TransitionToIA
        TransitionToPrimaryStorageClass: !Ref TransitionToPrimaryStorageClass
        Encrypted: !Ref Encrypted
        KmsKeyId: !Ref KmsKeyId
      TemplateURL: !Ref TemplateURL

  AccessPoint:
    Type: 'AWS::EFS::AccessPoint'
    Properties:
      FileSystemId: !GetAtt Efs.Outputs.Efs
      # Give full access. "0" is root user
      PosixUser:
        Gid: 0
        Uid: 0
      RootDirectory:
        Path: /    
  
  Ec2Instance: 
    Type: AWS::EC2::Instance
    Properties: 
      LaunchTemplate:
        LaunchTemplateId: !Ref LaunchTemplate    
        Version: !GetAtt LaunchTemplate.LatestVersionNumber
      SubnetId: !Ref Subnet
  
    
```
