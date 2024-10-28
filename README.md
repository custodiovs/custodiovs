# PROJETO DOCKER E WORDPRESS NA AWS

### CRIAR VPC E SUBNETS
- Na aba VPC, vá até Suas VPCs > Criar VPC;
- Coloque em VPC e muito mais;
- Dê um nome para a sua VPC;
- Coloque o IPV4 CIDR da sua preferência;
- Crie com 2 Az, com 2 públicas e 2 privadas;
### GRUPOS DE SEGURANÇA
1. SG - EC2
- Regras de entrada
   -  HTTP (80) - Qualquer endereço IPV4
     - SSH (22) - MEU IP
- Regras de saída
   - Todo o tráfego (0.0.0.0)
2. SG - RDS
- Regras de entrada
   - MySQL/AURORA (3306) - Grupo de Origem (SG - EC2)
- Regras de saída
   - Todo o tráfego (0.0.0.0)
3. SG - EFS
- Regras de entrada
   - NFS (2049) - Grupo de Origem (SG - EC2)
- Regras de saída
   - Todo o tráfego (0.0.0.0)
4. SG - ALB
- Regras de entrada
   - HTTP (80) -  Qualquer endereço IPV4
   - HTTPS (443) - Qualquer endereço IPV4
- Regras de saída
   - Todo o tráfego (0.0.0.0)
### CRIANDO O RDS
- Vá até RDS > Banco de dados > Criar Banco de dados;
- Criação Padrão;
- Opções de mecanismo > MySQL;
- Modelos > Nível gratuito;
- Configurações > Nome do BD > Autogerenciada > Senha do BD;
- Configuração da Instância > db.t3.micro;
- Conectividade > Não se conectar em nenhum recurso de computação do EC2 > IPV4 > Sua VPC > Grupo de Segurança Existente (SG-RDS) > Sem Preferência;
### CRIANDO O EFS
- Vá até o EFS > Criar sistema de arquivos;
- Coloque o nome e sua VPC > Personalizar;
- Em Rede > Colocar as sub-redes públicas que irão conter as Instâncias EC2 > Grupo de Segurança (SG -EFS);
- Criar
### CRIANDO AS INSTÂNCIAS
- Vá até EC2 > Executar Instância;
- Colocar as Tags desejadas;
- Selecionar a AMI do Sistema Operacional (Ubuntu) > Tipo t2.micro;
- Configurações de rede
   - Sua VPC
   - Sub-Rede Pública 1A (Primeira instância)
   - Sub-Rede Pública 1B (Segunda instaância)
- Detalhes avançados > Dados do usuário > Colocar o Script de inicialização que contém Docker, Docker-Compose e a Montagem do NFS;
```
#!/bin/bash

# Atualizar pacotes e instalar Docker e NFS
sudo apt update -y
sudo apt install -y docker.io nfs-common

# Iniciar e habilitar Docker
sudo systemctl start docker
sudo systemctl enable docker

# Montar o sistema de arquivos EFS
sudo mkdir /mnt/efs
sudo mount -t nfs4 -o nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2,noresvport (<seu efs id, que fica na aba anexar>) /mnt/efs

# Instalar Docker Compose
sudo curl -L "https://github.com/docker/compose/releases/download/1.29.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose

# Arquivos Docker Compose, Wordpress e MySQL para o RDS
sudo tee /mnt/efs/docker-compose.yml > /dev/null <<EOF
version: '3.1'

services:

  wordpress:
    image: wordpress:latest
    restart: always
    ports:
      - 80:80
    environment:
      WORDPRESS_DB_HOST: (seu endpoist do rds)
      WORDPRESS_DB_USER: (nome do seu rds)
      WORDPRESS_DB_PASSWORD: (senha do seu rds)
      WORDPRESS_DB_NAME: wordpress
    volumes:
      - /mnt/efs:/var/www/html

  db:
    image: mysql:latest
    restart: always
    environment:
      MYSQL_DATABASE: wordpress
      MYSQL_USER: (nome do seu rds)
      MYSQL_PASSWORD: (senha do seu rds)
      MYSQL_RANDOM_ROOT_PASSWORD: '1'
    volumes:
      - db:/var/lib/mysql

volumes:
  wordpress:
  db:
EOF

if ! grep -q "(<seu efs id):" /etc/fstab; then
    echo "(<seu efs id>)/ /mnt/efs nfs4 nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2,noresvport 0 0" | sudo tee -a /etc/fstab
fi

# Iniciar o Docker Compose
cd /mnt/efs
sudo docker-compose up -d
```
### CRIANDO MODELO DE EXECUÇÃO
- Em instâncias > Botão direito na Instância criada > Imagens e Modelo > Criar modelo a partir desta instância;
- Nome do seu template;
- Configurar passo a passo igual a instância EC2 anterior;
### CRIANDO GRUPO DE DESTINO
- Vá até EC2 > Grupos de Destino > Criar Grupos de Destino;
- Configuração Básica > Instância > Nome do seu GD > HTTP 80 > IPV4 > Sua VPC;
- Incluir as intâncias criadas > Criar;
### CRIANDO LOAD BALANCER
- Em EC2 > Load Balancer > Criar Load Balancer;
- Tipo > APLICAÇÃO DE LOAD BALANCER > Nome do seu ALB >
- Voltado para internet > Sua VPC > Selecionar as 2 AZ > Grupo de Segurança (SG-ALB) > PORTA HTTP 80 > Seu Grupo de destino;
### CRIANDO AUTO SCALLING
- Na aba EC2 > Auto Scalling > Criar Auto Scalling > Nome do seu AS >
- Selecionar o template criado no modelo de execução > Sua VPC > as 2 Sub-Redes > Anexar em um ALB existente > Seu Grupo de Destino > Tamanho do grupo 2, min 2 e max 4;
- Sem política. 
