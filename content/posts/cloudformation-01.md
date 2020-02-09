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

O primeiro passo é instalarmos a AWS CLI. Neste exemplo vamos demonstrar como instalar em um ambiente Linux (Ubuntu), mas caso você utilize Windows, basta instalar o WSL(e o Ubuntu pela Microsoft Store) e depois seguir o tutorial por ele.

 ```bash
sudo apt install awscli
 ```

 Depois precisamos configurar a awscli. Neste ponto você precisa ter uma Access Key e um Access Key ID. Basta criar um usuário no IAM como Programmatic access, que ao final da tela de criação serão exibidos estes dados (salve-os com carinho, pois não será possível resgatar a chave de acesso novamente para este usuário).

 ```bash
aws configure

AWS Access Key ID [None]:
AWS Secret Access Key [None]:
Default region name [None]:us-east-1
Default output format [None]: 
 ```
Obviamente eu ocultei minhas chaves de acesso, não quer ninguém minerando bitcoin na minha conta.

Feito isto, vamos entender o básico de um template do Cloudformation. É possível criar nossos arquivos usando yaml ou json, irei ilustrar em yaml para um entendimento mais fácil.

```yaml

```