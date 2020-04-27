---
title: "Cloudformation 03 - Recursos dentro da VPC e EC2 avançado"
date: 2020-04-27T16:08:38-03:00
draft: true
---

## Criando recursos dentro da VPC

Em nosso [artigo passado](https://fabio.monster/posts/cloudformation-02/) criamos nossa VPC completa, portanto antes de iniciar a criação dos recursos citados aqui, precisamos rodar aquela stack para criar toda a estrutura da VPC.

## Criando a EC2

Para criar nossa EC2, não tem segredo:

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
      GroupDescription: Enable SSH access via port 22
      SecurityGroupIngress:
      - IpProtocol: tcp
        FromPort: 22
        ToPort: 22
        CidrIp: "0.0.0.0/0"
```

Note que estamos referênciando a nossa VPC e a subnet pública.

## Atribuindo um Elastic IP

Sempre que reiniciamos uma instância EC2, o IP público padrão dela muda, pois é atribuído dinâmicamente.

Para contornar esta situação e criar registros DNS confiáveis, precisamos atribuir um Elastic IP para a nossa instância.

```yaml

  MyEIP:
      Type: AWS::EC2::EIP
      Properties:
        InstanceId: !Ref EC2Instance
        Domain: !GetAtt 'myVPC'

```

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
```



## Referências

[]()

[]()
