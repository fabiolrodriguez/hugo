---
title: "Cloudformation 04 - Nested stacks"
date: 2020-10-25T15:25:38-03:00
draft: false
---

## Executando diversas stacks

Depois de algum tempo volto aqui para conversarmos mais um pouco sobre Cloudformation.

Nos artigos anteriores criamos juntos recursos básicos como a nossa VPC e uma instância EC2 dentro desta VPC.

Agora iremos aprimorar estes scripts, criando VPC, EC2, RDS e instalar um Wordpress nesta estrutura, tudo com apenas um comando.

## Criando a nossa VPC

Como já citado anteriormente, a VPC é a nossa rede privada dentro da AWS, e é a base da nossa infra.

Todos os exemplos estão no github ao final do artigo, mas mostrarei os códigos fonte aqui para deixar a demostração mais detalhada.

```yaml
AWSTemplateFormatVersion: 2010-09-09
Description: Simple VPC example

Resources: 
  myVPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: 10.10.0.0/16
      EnableDnsSupport: true
      Tags:
        - Key: Name
          Value: MyVPC

  Pubsubnet:
    Type: AWS::EC2::Subnet
    Properties:
      AvailabilityZone: us-east-1a
      VpcId: !Ref myVPC
      CidrBlock: 10.10.1.0/24
      Tags:
        - Key: Name
          Value: Pubsubnet

  Pubsubnet2:
    Type: AWS::EC2::Subnet
    Properties:
      AvailabilityZone: us-east-1b
      VpcId: !Ref myVPC
      CidrBlock: 10.10.2.0/24
      Tags:
        - Key: Name
          Value: Pubsubnet2

  Privsubnet:
    Type: AWS::EC2::Subnet
    Properties:
      AvailabilityZone: us-east-1a
      VpcId: !Ref myVPC
      CidrBlock: 10.10.3.0/24
      Tags:
        - Key: Name
          Value: Privsubnet

  Privsubnet2:
    Type: AWS::EC2::Subnet
    Properties:
      AvailabilityZone: us-east-1b
      VpcId: !Ref myVPC
      CidrBlock: 10.10.4.0/24
      Tags:
        - Key: Name
          Value: Privsubnet2


  igwName:
    Type: AWS::EC2::InternetGateway
    Properties:
      Tags:
        - Key: Name
          Value: IG
  AttachGateway:
    Type: AWS::EC2::VPCGatewayAttachment
    Properties:
      VpcId: !Ref myVPC
      InternetGatewayId: !Ref igwName

  IGAcc:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref myVPC
      Tags:
        - Key: Name
          Value: IGAcc
  
  routeTableAssocIG:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref Pubsubnet
      RouteTableId: !Ref IGAcc

  routeTableAssocIG2:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref Pubsubnet2
      RouteTableId: !Ref IGAcc

  IntAcc:
    Type: AWS::EC2::Route
    Properties:
      RouteTableId: !Ref IGAcc
      DestinationCidrBlock: 0.0.0.0/0
      GatewayId: !Ref igwName

Outputs:
  VpcId:
      Description: ID of created VPC
      Value: !Ref myVPC
      Export:
        Name: VpcId
  
  PubsubnetId:
      Description: ID of public subnet
      Value: !Ref Pubsubnet
      Export:
        Name: PubsubnetId

  PubsubnetId2:
      Description: ID of public subnet
      Value: !Ref Pubsubnet2
      Export:
        Name: PubsubnetId2
  
  PrivsubnetId:
      Description: ID of private subnet
      Value: !Ref Privsubnet
      Export:
        Name: PrivsubnetId

  PrivsubnetId2:
      Description: ID of private subnet
      Value: !Ref Privsubnet2
      Export:
        Name: PrivsubnetId2
```

## Criando nossa instância de banco de dados RDS

Existem algumas maneiras de criar uma instância de banco de dados dentro da AWS, podendo ser inclusive uma instalação manual dentro de uma instância EC2.

Neste exemplo, vamos usar um banco de dados gerenciado, criando uma instância RDS free tier:

```yaml
AWSTemplateFormatVersion: 2010-09-09
Description: example template for MySQL RDS
Mappings:
  Config:
    Service:
      Name: wp-rds
Parameters:
  DBInstanceID:
    Default: wprds
    Description: My database instance
    Type: String
    MinLength: '1'
    MaxLength: '63'
    AllowedPattern: '[a-zA-Z][a-zA-Z0-9]*'
    ConstraintDescription: >-
      Must begin with a letter and must not end with a hyphen or contain two
      consecutive hyphens.
  DBName:
    Default: wordpress
    Description: My database
    Type: String
  DBUser:
    Default: 'admin'
    Type: String
  DBPassword:
    Default: 'myADM2029'
    Type: String
  DBidentifier: 
    Default: 'wordpress'
    Type: String
  DBInstanceClass:
    Default: db.t2.micro
    Description: DB instance class
    Type: String
    ConstraintDescription: Must select a valid DB instance type.
  DBAllocatedStorage:
    Default: '20'
    Description: The size of the database (GiB)
    Type: Number
    MinValue: '5'
    MaxValue: '1024'
    ConstraintDescription: must be between 20 and 65536 GiB.
  DBidentifier: 
    Default: 'wordpress'
    Type: String
Resources:
  DBSubnetGroup:
    Type: 'AWS::RDS::DBSubnetGroup'
    Properties:
      DBSubnetGroupDescription: 'Wordpress DB'     
      SubnetIds: 
        - !ImportValue PubsubnetId
        - !ImportValue PubsubnetId2
        - !ImportValue PrivsubnetId
        - !ImportValue PrivsubnetId2

  DatabaseSecurityGroup:
    Type: 'AWS::EC2::SecurityGroup'
    Properties:
      GroupDescription: 'SG DB'
      VpcId: !ImportValue VpcId
      SecurityGroupIngress:
      - IpProtocol: tcp
        FromPort: 3306
        ToPort: 3306
        # CidrIp: 0.0.0.0/0
        SourceSecurityGroupId: !ImportValue WPSG
    
  DBInstance: 
    DeletionPolicy: Snapshot
    Properties:
      DBSubnetGroupName: !Ref DBSubnetGroup
      AllocatedStorage: 
        Ref: DBAllocatedStorage
      DBInstanceClass: 
        Ref: DBInstanceClass
      DBName: 
        Ref: DBName
      Engine: mysql
      EngineVersion: "5.7"
      MasterUserPassword: 
        Ref: DBPassword
      MasterUsername: 
        Ref: DBUser
      DBInstanceIdentifier:
        Ref: DBidentifier
      PubliclyAccessible: "false"
      MultiAZ: "false"
      VPCSecurityGroups:
      - !Ref DatabaseSecurityGroup
    Type: "AWS::RDS::DBInstance"

Outputs:
  DBEndpoint:
    Description: 'The connection endpoint for the DB'
    Value: !GetAtt [DBInstance, Endpoint.Address]
    Export:
      Name: DBEndpoint
```
## Criando nossa instância EC2

Por fim, iremos subir nossa instância EC2 que irá hospedar o nosso Wordpress.

Notem no bloco **Userdata**, onde estão os comandos para baixar e descompactar a aplicação, assim como instalar e iniciar um servidor web Apache.

```yaml
AWSTemplateFormatVersion: 2010-09-09
Description: example template of EC2 features

Resources:

  InstanceSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Enable SSH access via port 22 and 80
      VpcId: !ImportValue 'VpcId'
      SecurityGroupIngress:
      - IpProtocol: tcp
        FromPort: 22
        ToPort: 22
        CidrIp: "0.0.0.0/0"
      - IpProtocol: tcp
        FromPort: 80
        ToPort: 80
        CidrIp: "0.0.0.0/0"

  EC2Instance: 
    Type: AWS::EC2::Instance
    Properties:
      InstanceType: "t2.micro"
      ImageId: "ami-062f7200baf2fa504"
      KeyName: "testeec2"
      BlockDeviceMappings: 
      - DeviceName: "/dev/sdm"
        Ebs: 
          VolumeType: "gp2"
          DeleteOnTermination: "true"
          VolumeSize: "8"
      NetworkInterfaces: 
        - AssociatePublicIpAddress: "true"
          DeviceIndex: "0"
          GroupSet: 
            - !GetAtt "InstanceSecurityGroup.GroupId"
          SubnetId: !ImportValue "PubsubnetId"
      UserData:
        Fn::Base64:
          !Sub |
            #!/bin/bash -xe
            yum -y update
            yum install httpd php php-mysql -y
            cd /var/www/html
            wget https://wordpress.org/wordpress-5.1.1.tar.gz
            tar -xzf wordpress-5.1.1.tar.gz
            cp -r wordpress/* /var/www/html/
            rm -rf wordpress
            rm -rf wordpress-5.1.1.tar.gz
            chmod -R 755 wp-content
            chown -R apache:apache wp-content
            systemctl start httpd
            systemctl enable httpd

  MyEIP:
    Type: AWS::EC2::EIP
    Properties:
      InstanceId: !Ref EC2Instance
      Domain: !ImportValue 'VpcId'
    
Outputs:
  PublicIP:
    Description: Public IP address of the newly created EC2 instance
    Value: !GetAtt [EC2Instance, PublicIp]
    Export:
        Name: PublicIP
  WPSG:
    Description: Security Group of Instance
    Value: !Ref InstanceSecurityGroup
    Export:
        Name: WPSG  
```


## Executando todas as stacks

Poderíamos executar cada uma destas stacks individualmente, desde que a criação da VPC fosse executada primeiro.

Mas existe uma maneira melhor e mais automatizada para isto, que são as **nested stacks**.

Basicamente, criamos um template do CloudFormation que irá chamar os demais templates. Neste caso, precisamos subir os templates individuais para um bucket S3.

Depois, referenciamos a URL de cada template no campo **TemplateURL**.

Por fim, utilizaremos o atributo **DependsOn** para garantir a execução dos templates na ordem correta. Primeiro a VPC, depois a EC2 e por fim o banco de dados.

```yaml
AWSTemplateFormatVersion: 2010-09-09
Description: Working with nested stacks

Resources: 

  VPC:
    Type: "AWS::CloudFormation::Stack"
    Properties:
      TemplateURL: https://cfplay.s3.amazonaws.com/vpc.yaml
      TimeoutInMinutes: 5

  EC2:
    Type: "AWS::CloudFormation::Stack"
    Properties:
      TemplateURL: https://cfplay.s3.amazonaws.com/ec2.yaml
      TimeoutInMinutes: 5
    DependsOn: VPC

  RDS:
    Type: "AWS::CloudFormation::Stack"
    Properties:
      TemplateURL: https://cfplay.s3.amazonaws.com/rds.yaml
      TimeoutInMinutes: 15
    DependsOn: EC2

```

Executamos a stack:

```bash
➜  wp (master) ✗ aws cloudformation deploy --stack-name wp --template-file wp.yaml
```
## Acessando a aplicação

Pelo console da AWS, acessamos os exports do CloudFormation.

Primeiro, precisamos do IP externo.

![https/://fabio.monster](https://fabio.monster/images/exports.png)

Agora basta abrir o navegador e colar este IP na barra de endereços para executar a configuração do seu Wordpress.

![https/://fabio.monster](https://fabio.monster/images/wp-config.png)

Ainda nos exports, copiar o endpoint do banco de dados:

![https/://fabio.monster](https://fabio.monster/images/dbendpoint.png)

Preencher as configurações utilizando os dados do nosso RDS:

![https/://fabio.monster](https://fabio.monster/images/wp-configured.png)

## Conclusão

Existem diversas melhorias possíveis, como adicionar um nome DNS para o nosso site e retirar a senha do banco de dados do template, armazenando de maneira segura este dado. Mas este é conteúdo para um próximo post.

Não se esqueçam de apagar tudo depois dos testes:

```bash
➜  wp (master) ✗ aws cloudformation delete-stack --stack-name wp
```

Todos os arquivos completos ficam no meu [Github](https://github.com/fabiolrodriguez/cloudformation-playground).

## Referências

[https://docs.aws.amazon.com/pt_br/AWSEC2/latest/UserGuide/user-data.html](https://docs.aws.amazon.com/pt_br/AWSEC2/latest/UserGuide/user-data.html)

[https://docs.aws.amazon.com/pt_br/AWSCloudFormation/latest/UserGuide/aws-properties-rds-database-instance.html](https://docs.aws.amazon.com/pt_br/AWSCloudFormation/latest/UserGuide/aws-properties-rds-database-instance.html)
