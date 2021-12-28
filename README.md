# aws-cloudformation-custom-efs
Custom EFS с автоматическим восстановлением из резервной копии

## Проблема в текущей реализации EFS
При полном восстановлении моментального снимка EFS в новую файловую систему файлы помещаются не в корень файловой системы, как вы могли бы ожидать, а в каталог с именем aws-backup-restore_datetime.
Это создает трудности. Вам нужно смонтировать EFS через EC2 или Lambda, переместить файлы и удалить этот каталог. Обсуждается здесь https://forums.aws.amazon.com/thread.jspa?threadID=346945
Кроме того AWS в настоящее время не поддерживает восстановление снимка состояния EFS из CloudFormation. Хотя, например, для RDS есть специальное свойство под названием DBSnapshotIdentifier, если вы его укажете, база данных будет восстановлена ​​автоматически. Было бы неплохо иметь подобное свойство для EFS

Это решение предлагает полностью исключить использование ресурса AWS::EFS::FileSystem в ваших шаблонах и заменить его на вложенный стек. 

 

## Синтаксис
Чтобы объявить эту сущность в шаблоне AWS CloudFormation, используйте следующий синтаксис:

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
```
> К сожалению на данный момент решение поддерживает не все свойства AWS::EFS::FileSystem которые вы можете найти здесь https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-efs-filesystem.html. Если вы не видите какого то свойства, значит оно не поддерживается или установлено в значение по умолчанию и не может быть изменено. Запрашивайте свойства в issues.

## Свойства

### BackupVaultName 
Название хранилища в котором хранится резервная копия

Обязательно: Да, если задан RecoveryPointArn

Тип: Строка

Для обновления требуется: Замена

### RecoveryPointArn 
Arn точки восстановления. Если этот параметр равен пустой строке, то будет создана пустая фаловая система.
Чтобы понять, как это работает и для чего предназначено свойство RecoveryPointArn, см. описание свойства DBSnapshotIdentifier в документации RDS. Поведение аналогично
https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-properties-rds-database-instance.html#cfn-rds-dbinstance-dbsnapshotidentifier

Обязательно: Нет

Тип: Строка

Для обновления требуется: Замена

### Vpc
Идентификатор VPC, например vpc-a123baa3. В этой VPC будет создана цель монтирования

Обязательно: Да

Тип: Строка

Для обновления требуется: Замена

### Subnet 
В этой подсети будет создана цель монтирования. Подсеть должна принадлежать указанной ранее VPC 

Обязательно: Да

Тип: Строка

Для обновления требуется: Замена

### SecurityGroup 
Группа безопасности которая используется целью монтирования. Группа безопасности должна принадлежать указанной ранее VPC 

Обязательно: Да

Тип: Строка

Для обновления требуется: Замена

## Возвращаемые значения

### Fn :: GetAtt
Внутренняя функция возвращает значение для указанного атрибута этого типа. Ниже приведены доступные атрибуты и примеры возвращаемых значений.

#### Outputs.Efs
Идентификатор файловой системы, например, fs-06799d63c3aeb4707


