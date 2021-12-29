# aws-cloudformation-custom-efs
Custom EFS с автоматическим восстановлением из резервной копии

## Проблема в текущей реализации EFS
При полном восстановлении моментального снимка EFS в новую файловую систему файлы помещаются не в корень файловой системы, как вы могли бы ожидать, а в каталог с именем aws-backup-restore_datetime.
Это создает трудности. Вам нужно смонтировать EFS через EC2 или Lambda, переместить файлы и удалить этот каталог. Обсуждается здесь https://forums.aws.amazon.com/thread.jspa?threadID=346945
Кроме того AWS в настоящее время не поддерживает восстановление снимка состояния EFS из CloudFormation. Хотя, например, для RDS есть специальное свойство под названием DBSnapshotIdentifier, если вы его укажете, база данных будет восстановлена ​​автоматически. Было бы неплохо иметь подобное свойство для EFS.
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

### TemplateURL 
Расположение файла, содержащего тело шаблона. URL-адрес должен указывать на шаблон расположенный в корзине Amazon S3. Скопируйте файл template.yaml в свою корзину и укажите здесь ссылку на него.

Обязательно: Да

Тип: Строка

Для обновления требуется: ?

### PerformanceMode
Режим производительности файловой системы. Мы рекомендуем generalPurpose режим производительности для большинства файловых систем. Файловые системы, использующие maxIOрежим производительности, могут масштабироваться до более высоких уровней совокупной пропускной способности и операций в секунду с компромиссом в виде немного более высоких задержек для большинства файловых операций. Режим производительности нельзя изменить после создания файловой системы.

Required: Нет

Type: String

Allowed values: generalPurpose | maxIO

Для обновления требуется: Замена

## Возвращаемые значения

### Fn :: GetAtt
Внутренняя функция возвращает значение для указанного атрибута этого типа. Ниже приведены доступные атрибуты и примеры возвращаемых значений.

#### Outputs.Efs
Идентификатор файловой системы, например, fs-06799d63c3aeb4707

## Примеры

Следующий образец шаблона монтирует файловую систему к EC2 инстансу. В любой момент вы можете изменить параметры шаблона BackupVaultName и RecoveryPointArn и тогда CloudFormation восстановит Вашу файловую систему из резервной копии, создаст новый инстанс EC2 и смонтирует к нему восстановленную файловую систему. Затем старая фаловая систему и старый инстанс благополучно удалятся. Великолепно, правда? =)

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
    Default: arn:aws:backup:eu-central-1:918404900336:recovery-point:8ce28d06-437d-4c1b-83fa-dc2e46e729fe
  TemplateURL:
    Type: String
    MinLength: 1   
    Default: https://s3.eu-central-1.amazonaws.com/aos.bucket/template.yaml
    Description: URL for template.yaml. Should be in S3 

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

