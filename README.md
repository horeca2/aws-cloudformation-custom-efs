# aws-cloudformation-custom-efs

Problem:
1. When fully restoring an EFS snapshot to a new filesystem, the files are not placed in the root of the filesystem as expected, but in a directory named aws-backup-restore_datetime
This creates difficulties. You need to mount EFS via EC2 or lambda, move files and delete this directory. Discussed here https://forums.aws.amazon.com/thread.jspa?threadID=346945

2. AWS does not currently support restoring an EFS snapshot from CloudFormation. Although, for example, for RDS there is a special property called DBSnapshotIdentifier, if you specify it, the database is automatically restored.
It would be nice to have a similar property for EFS

So there is no automation

This solution proposes to completely remove the use of the AWS :: EFS :: FileSystem resource in your templates and replace it with the "Custom :: EFS" resource.
In this case, the syntax will remain the same (or almost the same TODO: check !!!). Only the additional property SnapshotIdentifier is added

To understand how this works and what the SnapshotIdentifier property is for, see the description of the DBSnapshotIdentifier property in the RDS documentation. Behavior is similar
https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-properties-rds-database-instance.html#cfn-rds-dbinstance-dbsnapshotidentifier
