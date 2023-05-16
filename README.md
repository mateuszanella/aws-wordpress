# Aplicação WordPress na plataforma AWS.

<p> O presente documento faz parte da documentação de uma atividade do Programa de Bolsas da CompassUOL.
<p> A atividade consiste em: criar instâncias EC2 privadas que hosteiam uma aplicação do Wordpress, com integração de um banco de dados RDS, um grupo de Auto Scaling e um Load Balancer.

## Criação da VPC

Primeiramente, é necessária a criação de uma nova VPC com os seguintes atributos:
- Para esta tarefa, foi escolhido um range de CIDR de 512 IPs (198.162.0.0/23):
  - 198.162.1.0/24: para redes privadas;
  - 198.162.0.0/24: para redes públicas.
- Em seguida, é necessário configurar 4 subnets:
  - 2 privadas, para as instâncias que conterão a aplicação.
  - 2 públicas, para o Load Balancer.
- Após criadas as Subnets, é necessário conectá-las à internet, isso é feito conectando as redes Públicas à um Internet Gateway, e as privadas à um NAT Gateway.
  - Na tabela de rotas pública, cria-se uma rota com destino 0.0.0.0/0 para o Internet Gateway.
  - Na tabela de rotas privada, cria-se uma rota com destino 0.0.0.0/0 para o NAT Gateway.

## Criação dos Security Groups

Após a configuração da VPC, será necessário criar os respectivos Security Groups que serão utilizados, que serão os seguintes:
- Um grupo de segurança para o Bastion Host:
  - Liberar todo o tráfego (0.0.0.0/0) para a porta 22.
- Um grupo de segurança para o Load Balancer:
  - Liberar todo o tráfego (0.0.0.0/0) para a porta 80.
- Um grupo de seguraça para o Banco RDS:
  - Liberar a porta 3306 para o Grupo de Segurança Default.
- Um grupo de segurança para as Instâncias privadas:
  - Liberar o acesso à porta 22 para a Instância Bastion;
  - Liberar a porta 2049 (acesso do Elastic File System) para o Grupo de Segurança Default;
  - Liberar a porta 80 para o Grupo de Segurança do Load Balancer;
  - Liberar a porta 3306 para o Grupo de Segurança do Banco RDS.
> **Sobre liberar algumas portas para outros Grupos de Segurança:** isso é feito para garantir que a porta só será aberta para Instâncias que estejam atreladas à determinado Grupo de Segurança. É considerado uma boa prática, pois por mais que exista a possibilidade de utilizar o endereço de IP específico das máquinas EC2, também existe a chance do endereço mudar com a criação/destruição/reincialização de Instâncias.

----
- criar security groups
  - bastion - 22 - all
  - efs - 2049 - sg-default
  - db - 3306 - sg-default
  - instancias privadas
- criar efs
- criar banco rds
- criar ami da instancia
- criar launch template
- criar autoscaling group
- criar load balancer

