# aws-cloudformation-custom-efs


## Problem 1
When fully restoring an EFS snapshot to a new filesystem, the files are not placed in the root of the filesystem as expected, but in a directory named aws-backup-restore_datetime
This creates difficulties. You need to mount EFS via EC2 or lambda, move files and delete this directory. Discussed here https://forums.aws.amazon.com/thread.jspa?threadID=346945

If you only need to solve this problem use the following solution

### Installation:
Create a stack from a move-template.yaml. Leave the EFS field blank!

### Usage
1. Restore your recovery point to the new file system
2. Update the stack. Set the value of the EFS property to the id of the new file system
3. Run lambda function manually. Arn lambda functions can be found in the stack output (Or just click on it in the Resources tab)
4. Update the stack again, set the EFS property value to an empty string

EFS is ready. The stack can be reused!

How does it work:
The stack creates the lambda function and the required environment. 
If EFS is specified, the stack will create an access point and mount target for that EFS, and will attach EFS to the lambda function. The stack will return the ARN lambda function in output parameters.
If you call this lambda function, it will move the entire contents of the "aws-backup-restore_ *" folder to the root of the filesystem, and then delete this folder. A repeated call will of course result in an error.
On stack update when the EFS property is set to an empty string, the access point and mount target will be deleted from EFS, and EFS will also detach from the lambda. EFS is ready to use. 
Note! The stack is assumed to be used immediately after EFS is restored from a backup, so EFS should not have access points and mount targets set.

## Problem 2
AWS does not currently support restoring an EFS snapshot from CloudFormation. Although, for example, for RDS there is a special property called DBSnapshotIdentifier, if you specify it, the database is automatically restored.
It would be nice to have a similar property for EFS

This solution proposes to completely remove the use of the AWS :: EFS :: FileSystem resource in your templates and replace it with the "Custom :: EFS" resource.
In this case, the syntax will remain the same. Only the additional property RecoveryPoint is added

To understand how this works and what the RecoveryPoint property is for, see the description of the DBSnapshotIdentifier property in the RDS documentation. Behavior is similar
https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-properties-rds-database-instance.html#cfn-rds-dbinstance-dbsnapshotidentifier

### Installation:
1. Create a stack from a move-template.yaml. Leave the EFS field blank! In this solution, you do not need to use this stack manually! Forget about him
2. Create a stack from a template.yaml. Set the MoveStack property to the name of the stack you created in the previous step