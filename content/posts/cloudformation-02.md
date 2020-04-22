---
title: "Cloudformation 02 - VPC e Referências"
date: 2020-04-22T11:06:38-03:00
draft: true
---

## Evoluindo no Cloudformation

No primeiro artigo desta série, aprendemos como funciona o Cloudformation, configuramos o ambiente ecriamos nosso primeiro template.

Naquele exemplo, utilizamos a VPC padrão da AWS, o que não é algo recomendável. O melhor cenário é sempre criarmos a nossa própria VPC, com a faixa de rede que desejamos. Isto é importante para, por exemplo, utilizar uma faixa de rede diferente do seu datacenter atual e poder fechar uma VPN.

## Entendendo a VPC

Uma VPC é a sua nuvem privada virtual dentro da AWS (por isso a sigla VPC). É dentro da VPC que ficarão as nossas subnets, gateways, tabelas de roteamento e endpoints, de maneira bem semelhante com uma rede tradicional.

![https/://fabio.monster](https://fabio.monster/images/vpc.png)

Seguindo uma topologia bem simples e padrão, vamos criar os seguintes recursos:

* VPC
* Subnet pública
* Subnet privada
* Tabela de roteamento
* Associação a tabela de roteamento
* Internet gateway
* Rota para o Internet Gateway

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

Nos próximos passos, iremos utilizar bastante as referências, portanto precisamos entender esse ponto.

## Utilizando as Referências

Para referenciarmos um recurso previamente criado pelo Cloudformation, no mesmo template, utilizamos a tag:

```yaml
!Ref Recurso
```
Também é possível acessar recursos criados por outras stacks do Cloudformation, desde que sejam gravados nos Outputs.

Para referênciar um output, precisamos utilizar a tag:

```yaml
!GetAtt Recurso
```
Utilizaríamos esta opção, por exemplo, para criar recursos futuros dentro desta mesma VPC.

Para entender melhor, vamos fazer na prática.

## Criando as subnets

Vamos criar as subnets pública e privada, com as respectivas faixas de IP 10.10.1.0/24 e 10.10.2.0/24.

```yaml

Pubsubnet:
  Type: AWS::EC2::Subnet
  Properties:
    AvailabilityZone: us-east-1
    VpcId: !Ref myVPC
    CidrBlock: 10.10.1.0/24
    Tags:
      - Key: Name
        Value: Pubsubnet

Privsubnet:
  Type: AWS::EC2::Subnet
  Properties:
    AvailabilityZone: us-east-1
    VpcId: !Ref myVPC
    CidrBlock: 10.10.2.0/24
    Tags:
      - Key: Name
        Value: Privsubnet

```
Note como referênciamos nossa VPC como ```yaml !Ref myVPC ```.

## Criando o Internet Gateway

Agora iremos criar e anexar nosso gateway de Internet, também usando as referências.

```yaml
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
```
## Criado as tabelas e rotas

Por fim, criamos a tabela de roteamento, associamos a subnet pública e criamos a rota para o IGW:

```yaml

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

  IntAcc:
    Type: AWS::EC2::Route
    Properties:
      RouteTableId: !Ref IGAcc
      DestinationCidrBlock: 0.0.0.0/0
      GatewayId: !Ref igwName

```

## Rodando a stack

Agora basta rodar o nosso template via awscli:

```bash
aws cloudformation deploy --stack-name vpctest --template-file vpc.yaml
```

## Referências

https://docs.aws.amazon.com/pt_br/vpc/latest/userguide/how-it-works.html
