# aws-cloudformation-custom-efs

Problem:
1. When fully restoring an EFS snapshot to a new filesystem, the files are not placed in the root of the filesystem as expected, but in a directory named aws-backup-restore_datetime
This creates difficulties. You need to mount EFS via EC2 or lambda, move files and delete this directory. Discussed here https://forums.aws.amazon.com/thread.jspa?threadID=346945

2. AWS does not currently support restoring an EFS snapshot from CloudFormation. Although, for example, for RDS there is a special property called DBSnapshotIdentifier, if you specify it, the database is automatically restored.
It would be nice to have a similar property for EFS

So there is no automation

В этом решении предлагается полность отказаться от использования ресурса AWS::EFS::FileSystem в ваших шаблонах и заменить его на ресурс "Custom::EFS"
При этом синтаксис останется прежним (или почти прежним TODO: проверить!!!). Добавляется только дополнительное свойство SnapshotIdentifier

Чтобы понять как это работает и для чего свойство SnapshotIdentifier посмотрите описание свойства DBSnapshotIdentifier в документации на RDS. Поведение аналогично
https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-properties-rds-database-instance.html#cfn-rds-dbinstance-dbsnapshotidentifier
