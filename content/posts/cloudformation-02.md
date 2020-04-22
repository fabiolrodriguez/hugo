---
title: "Cloudformation 02 - VPC e Referências"
date: 2020-04-22T11:06:38-03:00
draft: true
---

## Evoluindo no Cloudformation

No primeiro artigo desta série, aprendemos como funciona o Cloudformation, configuramos o ambiente ecriamos nosso primeiro template.

Naquele exemplo, utilizamos a VPC padrão da AWS, o que não é algo recomendável. O melhor cenário é sempre criarmos a nossa própria VPC, com a faixa de rede que desejamos. Isto é importante para, por exemplo, utilizar uma faixa de rede diferente do seu datacenter atual e poder fechar uma VPN.

## Criando nossa primeira VPC

Criar uma VPC utilizando o Cloudformation é bastante simples, seguimos o exemplo:

```yaml
Resources: 
  myVPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: 10.10.0.0/16
      EnableDnsSupport: true
      Tags:
        - Key: Name
          Value: MyVPC
```
Podemos abservar que a faixa de rede utilizada é a 10.10.0.0/16. Neste caso a escolha foi aleatória, mas o correto é verificar a sua necessidade para evitar problemas futuros.

## Utilizando as Referências

Feito isto, vamos adicionar ao nosso template uma instância EC2 dentro da nossa VPC, referenciando o recurso criado anteriormente.

```yaml
Resources:
  EC2Instance: 
    Type: AWS::EC2::Instance
    Properties:
      InstanceType: "t2.micro"
      SecurityGroups: [!Ref 'InstanceSecurityGroup']
      ImageId: "ami-062f7200baf2fa504"
      KeyName: "teste"
      VpcId: !Ref 'myVPC'
      BlockDeviceMappings: 
      - DeviceName: "/dev/sdm"
        Ebs: 
          VolumeType: "gp2"
          DeleteOnTermination: "true"
          VolumeSize: "8"
  InstanceSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Enable SSH access via port 22
      SecurityGroupIngress:
      VpcId: !Ref 'myVPC'
      - IpProtocol: tcp
        FromPort: 22
        ToPort: 22
        CidrIp: "0.0.0.0/0"
```
## Criando os recursos

Agora basta rodar o nosso template via awscli:

```bash
aws cloudformation deploy --stack-name vpctest --template-file vpc.yaml
```