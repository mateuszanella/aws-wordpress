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
> **Sobre liberar algumas portas para outros Grupos de Segurança:** isso é feito para garantir que a porta só será aberta para Instâncias que estejam atreladas à determinado Grupo de Segurança. É considerado uma boa prática, pois por mais que exista a possibilidade de utilizar o endereço de IP específico das máquinas EC2, também existe a chance do endereço mudar com a criação/destruição/reincialização das Instâncias.

## Criação do EFS
  
- Para esta atividade, será utilizado um EFS para armazenar os estáticos do container do Wordpress.
  
## Criação do Banco RDS
  
- Após a criação do EFS, é criado também um banco RDS do tipo MySQL. Durante a sua criação, é necessária atenção à alguns pontos:
  - A configuração do Master User e de sua senha é extremamente importante para a configuração do Wordpress;
  - O RDS é configurado na VPC criada para a atividade, e recebe o seu respectivo Grupo de Segurança;
  - É recomendável definir um banco de dados inicial, para facilitar a configuração do Wordpress.
  
## Configuração da Instância e criação da AMI
  
Existem diversos métodos de fazer a configuração de uma instância para uma atividade como esta, porém, nesse projeto, será feito via o uso de imagens.
  
### Inicialização da Instância e do EFS
  
- Primeiramente, cria-se uma instância base, que será o molde para a imagem que será utilizada para o Launch Template.
- A instância pode ser iniciada com o seguinte user data, que instala os serviços de EFS da AWS, o serviço do Docker e o Docker Compose:

```
#!/bin/bash
yum update -y
yum install -y amazon-efs-utils
yum -y install docker
service docker start
usermod -a -G docker ec2-user
chkconfig docker on
curl -L https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m) -o /usr/local/bin/docker-compose
chmod +x /usr/local/bin/docker-compose
reboot
```

- Com isso feito, todos esses serviços já devem estar montados.
- Em seguida, podemos configurar a montagem do EFS no arquivo ```/etc/fstab```.
- Podemos fazer isso por meio do seguinte comando:
<p>```sudo echo '[id-do-filesystem].efs.[regiao].amazonaws.com:/ /efs nfs4 defaults,_netdev 0 0' >> /etc/fstab```
- Em seguida, podemos criar um documento do docker-compose para criar um container com a aplicação Wordpress.

### Criação do Container da aplicação Wordpress  

- Cria-se uma pasta com um nome qualquer.
- Dentro dessa pasta, será criado um arquivo ```docker-compose.yml```.
- O arquivo deve conter as seguintes informações:
```
version: '3'
services:
  wordpress:
    image: wordpress:latest
    ports:
      - 80:80
    restart: always
    environment:
      - WORDPRESS_DB_HOST=<Link do servidor RDS>
      - WORDPRESS_DB_USER=<Master User>
      - WORDPRESS_DB_PASSWORD=<Senha do Master User>
      - WORDPRESS_DB_NAME=<Nome da DataBase>
    volumes:
      - /efs/wp_data:/var/www/html
  db:
    image: mysql:8.0.27
    volumes:
      - /efs/db_data:/var/lib/mysql
    restart: always
    environment:
      - MYSQL_DATABASE=<Nome da DataBase>
      - MYSQL_USER=<Master User>
      - MYSQL_PASSWORD=<Senha do Master User>
      - MYSQL_ROOT_PASSWORD=<Senha do Master User>
      - MYSQL_HOST=<Link do servidor RDS>
      - MYSQL_PORT=3306
volumes:
  wp_data:
  db_data:
```
- O arquivo docker-compose deve criar um container com o MySQL (já configurado com o Banco RDS criado anteriormente) e com o Wordpress.
- Criado o arquivo, basta digitar o seguinte comando para baixar e executar os containers:
```
docker-compose up -d  
```

###  Criação da AMI

- Feito isso, podemos partir para a criação da imagem.
- Com o botão direito, clique na instância em questão, e vá para ```Image and templates``` > ```Create image```.
  
## Criação do Launch Template
  
> Nesse caso, é possível selecionar a opção ```Auto Scaling guidance``` para ajudar na criação do Launch Template.
  
- Primeiramente, será selecionada a AMI feita anteriormente.
- Em seguida, é feita a escolha da instância desejada, nesse caso, utilizaremos uma ```t2.micro``` para fins de demonstração.
- Na aba de subnets, não inclua nenhuma neste Template, pois isso será definido na criação do Auto Scaling Group.
- Selecione dois Grupos de Segurança: o criado especificamente para as Instâncias Privadas e o grupo Default (**IMPORTANTE**).
> É de extrema importância lembrar de adicionar o grupo Default de segurança, pois mesmo que ele não possua nenhuma regra de entrada, é vital para o funcionamento das regras de outros Grupos de Segurança.
- Com isso, está completa a criação do Launch Template.
  
----


- criar autoscaling group
- criar load balancer

