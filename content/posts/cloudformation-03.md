---
title: "Cloudformation 03 - Recursos dentro da VPC e EC2 avançado"
date: 2020-07-01T14:08:38-03:00
draft: false
---

## Criando recursos dentro da VPC

Em nosso [artigo passado](https://fabio.monster/posts/cloudformation-02/) criamos nossa VPC completa. Portanto antes de iniciar a criação dos recursos citados aqui, precisamos rodar aquela stack para criar toda a estrutura da VPC.

Você pode encontrar o template cloudformation completo no meu [github](https://github.com/fabiolrodriguez/cloudformation-playground/tree/master/vpc).

## Criando a EC2

Vamos criar uma instância EC2 dentro da nossa VPC previamente criada, usando o seguinte template:

```yaml

Resources:
  EC2Instance: 
    Type: AWS::EC2::Instance
    Properties:
      InstanceType: "t2.micro"
      SecurityGroups: [!Ref 'InstanceSecurityGroup']
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
          SubnetId: !GetAtt "Pubsubnet"

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
```

Note que estamos referênciando a nossa VPC na criação do Security group e a subnet pública na instância EC2.

## Atribuindo um Elastic IP

Sempre que reiniciamos uma instância EC2, o IP público padrão dela muda, pois é atribuído dinâmicamente.

Para resolver esta situação e criar registros DNS confiáveis, precisamos atribuir um Elastic IP para a nossa instância.

```yaml

  MyEIP:
      Type: AWS::EC2::EIP
      Properties:
        InstanceId: !Ref EC2Instance
        Domain: !GetAtt 'myVPC'

```
Novamente utilizamos nossa referência a VPC e a instância EC2 criada no exemplo anterior.

## Utilizando o userdata

Durante a primeira inicialização de nossa instância EC2, podemos executar uma série de comandos. Isto é muito útil para automação de ambientes e dispensar o uso de mais ferramentas, como o Ansible por exemplo.

Basta adicionar o bloco do userdata logo depois da nossa placa de rede:

```yaml
UserData:
        Fn::Base64:
          !Sub |
            #!/bin/bash -xe
            yum -y update
            yum install httpd -y
            systemctl start httpd
            systemctl enable httpd
```

Neste exemplo, estamos instalando e iniciando um servidor apache.

Vamos salvar nosso IP público nos outputs para uso futuro:

```yaml
Outputs:
  PublicIP:
    Description: Public IP address of the newly created EC2 instance
    Value: !GetAtt [EC2Instance, PublicIp]
    Export:
        Name: PublicIP
```

## Rodando a stack

Agora basta rodar o nosso template via awscli:

```bash
➜  ec2adv (master) ✗ aws cloudformation deploy --stack-name ec2adv --template-file ec2.yaml

Waiting for changeset to be created..
Waiting for stack create/update to complete
Successfully created/updated stack - ec2adv
```

## Conclusão

Neste artigo aprendemos como criar recursos dentro da nossa VPC de maneira automatizada, bem como rodar comandos em nossa EC2 durante a criação.

Não se esqueçam de destruir os recursos para não serem cobrados até o próximo passo do tutorial.

Todos os arquivos completos ficam no meu [Github](https://github.com/fabiolrodriguez/cloudformation-playground).

## Referências

[https://docs.aws.amazon.com/pt_br/AWSEC2/latest/UserGuide/user-data.html](https://docs.aws.amazon.com/pt_br/AWSEC2/latest/UserGuide/user-data.html)

