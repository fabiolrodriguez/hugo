---
title: "Iniciando com o AWS Cloudformation"
date: 2020-02-08T17:17:38-03:00
draft: true
---

## O que é Cloudformatiom ?

O Cloudformation é a ferramenta da própria AWS que possibilita o provisionamento e gerenciamento da nossa infraestrutura como código, ou apenas IaaC. 

Atualmente estou trabalhando em um projeto inteiramente gerido via Cloudformation, então decidi compartilhar um pouco do que venho estudando e aprendendo.

## Porque Cloudformatiom?

Sabemos que a ferramenta de IaaC mais popular do mercado é o Terraform da Hashicorp, e com louvor. 

O Terraform geralmente é mais vantajoso pois podemos escrever a infra como código para diversas clouds diferentes (e até para o VMWare), enquanto o Cloudformation é exclusivo para a AWS, obviamente.

A Vantagem no caso do Cloudformation é que não precisamos gerenciar o estado da nossa stack (o popular tfstate). Não é complexo ter um gerenciamento de estado decente no Terraform, mas se mal feito, pode ser um grande problema. O Cloudformation vai simplesmente ler toda a infra provisionada por ele e aplicar as alterações.

Se sua stack é full AWS, porque não dar uma chance ao Cloudfomartion também?

## Instalação e configuração

### AWS Console

#### Criar uma Keypair

Para simplificar o tutorial, vamos criar apenas um único recurso via console, que será a chave de acesso ssh.

Acessando *Services -> EC2*, no menu lateral, clique em Keypairs:

![https/://fabio.monster](https://fabio.monster/images/keypairs.png)

Clique em *Create key pair* para criar a nova chave:

![https/://fabio.monster](https://fabio.monster/images/created-keypair.png)

Preencha o nome como *teste*, selecione o formato *pem* e clique me *Create key pair*.

![https/://fabio.monster](https://fabio.monster/images/keypair-done.png)

#### Criar um usuário.

Acesse *Services -> IAM*.

No meu lateral, na parte de *Access Management* clique em *Users*.

![https/://fabio.monster](https://fabio.monster/images/iam.png)

Depois clique em *Add User*

![https/://fabio.monster](https://fabio.monster/images/add-user.png)

Defina um nome para a conta e o tipo como *Programmatic access*.

![https/://fabio.monster](https://fabio.monster/images/programmatic.png)

Clique *Next* para definir as permissões.

Selecione a aba *Attach existing policies directly* e marque a caixa de seleção *Administrator Access*.

![https/://fabio.monster](https://fabio.monster/images/admacc.png)

Clique em *Next* para definir uma TAG, depois *Next* novamente na tela de *Review*.

![https/://fabio.monster](https://fabio.monster/images/user.png)

A tela seguinte irá exibir a sua *Access Key* e o seu *Access Key ID*, recomendo fortemente **salvar estes dados**, pois não é possível rever a chave de acesso de um usuário já criado.

Agora chega de usar o console, todo o resto faremos pela *AWS CLI*!

### AWS CLI

O primeiro passo é instalarmos a AWS CLI. Neste exemplo vamos demonstrar como instalar em um ambiente Linux (Ubuntu), mas caso você utilize Windows, basta instalar o WSL(e o Ubuntu pela Microsoft Store) e depois seguir o tutorial por ele.

 ```bash
sudo apt install awscli
 ```

 Depois precisamos configurar a awscli. Neste ponto você precisa ter a Access Key e o Access Key ID do usuário que criamos anteriormente.

 ```bash
aws configure

AWS Access Key ID [None]:
AWS Secret Access Key [None]:
Default region name [None]:us-east-1
Default output format [None]: 
 ```
Obviamente eu ocultei minhas chaves de acesso, não quer ninguém minerando bitcoin na minha conta.

## Primeiro template

Feito isto, vamos entender o básico de um template do Cloudformation. É possível criar nossos arquivos usando *yaml* ou *json*, irei ilustrar em yaml para um entendimento mais fácil.

```yaml
Resources:
  EC2Instance: 
    Type: AWS::EC2::Instance
    Properties:
      InstanceType: "t2.micro"
      SecurityGroups: [!Ref 'InstanceSecurityGroup']
      ImageId: "ami-062f7200baf2fa504"
      KeyName: "teste"
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
      - IpProtocol: tcp
        FromPort: 22
        ToPort: 22
        CidrIp: "0.0.0.0/0"
```
Sempre é necessário pelo menos um resource para o template. Neste caso, temos 2, uma instância EC2 e um security group para a mesma. Estamos liberando o acesso SSH para toda a internet neste exemplo, o que não é nem de longe recomendado.

A configuração da nossa instância ficou a seguinte:

* O tipo da instância é t2.micro
* imagem base: ami-062f7200baf2fa504
* Um volume EBS do tipo gp2 com 8 GB de tamanho.
* Utilizando a keypair chamada 'teste'.

Estes recursos serão criados dentro da nossa *VPC* default.

## Utilizando a awscli para gerenciar o Cloudformation

Existem duas maneiras de executar um template do cloudformation na AWS, via cli ou via console (também podemos utilizar webhooks integrados ao git, mas fica para outro artigo). 

Neste caso, vamos utilizar a awscli para fazer o deploy do nosso template.

```bash
aws cloudformation deploy --stack-name ec2test --template-file ec2.yaml

Waiting for changeset to be created..
Waiting for stack create/update to complete
Successfully created/updated stack - ec2test
```

É possível acompanhar a criação da stack pelo console da AWS também, mesmo quando criamos via cli.

Agora basta verificar no seu console sua instância criada, em *Services -> EC2*. 

Copie o IP público da sua instância para realizar o teste de conexão.

Lembra da Keypair que criamos via console? vamos utilizá-la para conectar na instância via ssh.

```bash
cp ~/Downloads/teste.pem ~/.ssh/teste.pem
chmod 400 ~/.ssh/teste.pem
```

Conectar via SSH:

```bash
ssh ec2-user@18.212.135.252 -i ~/.ssh/teste.pem


       __|  __|_  )
       _|  (     /   Amazon Linux 2 AMI
      ___|\___|___|

https://aws.amazon.com/amazon-linux-2/
7 package(s) needed for security, out of 39 available
Run "sudo yum update" to apply all updates.
[ec2-user@ip-172-31-34-133 ~]$
```

Para apagar a stack:

```bash
aws cloudformation delete-stack --stack-name ec2test
```

Agora você não tem mais recursos criados na AWS, e não irá receber nenhuma conta surpresa no seu cartão de crédito.

Um grande benefício da IaaC para quem está estudando é exatamente isso, pode destruir todos os recursos após os testes e evitar cobranças indesejadas.

Lembrando que este foi um exemplo extremamente básico. Nos próximos artigos, iremos explorar um pouco mais os recursos do Cloudformation e deixar nossos templates mais elegantes.

Lembrando que todos os templates citados aqui também estão disponíveis no meu [Github](https://github.com/fabiolrodriguez/cloudformation-playground). Aproveita e também me segue no [Twitter](https://twitter.com/fabiolrodriguez) para saber quando eu publicar novos artigos.