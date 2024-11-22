# **Índice**
1. [Introdução](#introdução)
2. [Configuração da AWS](#configuração-da-aws)
3. [Configuração do Banco de Dados (MySQL)](#configuração-do-banco-de-dados-mysql)
4. [Configuração dos Servidores Backend](#configuração-dos-servidores-backend)
5. [Configuração do Servidor Web/Proxy](#configuração-do-servidor-webproxy)
    - 5.1. [Servindo o Frontend](#servindo-o-frontend)
    - 5.2. [Configuração do Proxy Reverso (usando nginx)](#configuração-do-proxy-reverso-usando-nginx)
    - 5.3. [Configuração do VPN (usando OpenVPN)](#configuração-do-vpn-usando-openvpn)
    - 5.4. [Atualizar as Regras de Segurança no AWS](#atualizar-as-regras-de-segurança-no-aws)
    - 5.5. [Como se conectar ao servidor VPN usando o OpenVPN GUI no Windows](#como-se-conectar-ao-servidor-vpn-usando-o-openvpn-gui-no-windows)
6. [(Opcional) - Adicionar certificação ao OpenVPN usando EasyRSA](#opcional---adicionar-certificação-ao-openvpn-usando-easyrsa)
7. [(Opcional) - Adicionar Health Check ao Load Balancer no Nginx](#opcional---adicionar-health-check-ao-load-balancer-no-nginx)

# Introdução
Este projeto implementa uma arquitetura de rede escalável usando instâncias da AWS. O objetivo é criar um ambiente robusto que suporte aplicações modernas, utilizando Docker para a contêinerização e Nginx para gerenciamento de tráfego. Ao seguir este guia, você aprenderá a configurar servidores backend, um banco de dados MySQL e um servidor web com balanceamento de carga.


<img src="https://github.com/cauancesar/Redes-Synergy/blob/main/imgs/_TopologiaRedes.drawio.png" height=800></img>


# Configuração da AWS

### 1. Criar uma instância EC2 para cada máquina:
1. Acesse o console do Amazon EC2.
2. Clique em "Executar Instância".
3. De um nome a instância.
4. Em "Imagens de aplicação e de sistema operacional (imagem de máquina da Amazon)" clique em Ubuntu.
5. Em "Par de chaves (login)" crie e selecione um novo par de chaves para acessar a máquina.
6. Em "Configurações de rede" selecione as opções "Permitir tráfego HTTP da Internet" e "Permitir tráfego SSH de".
7. (Opcional) - Em "Configurar armazenamento" aumento o tamanho do armazenamento caso necessário.
8. Clique em "Executar Instância".

### 2. Alocar um IP elástico às instâncias:
1. No console do Amazon EC2, no painel de navegação, clique em “IPs elásticos”.
2. Clique em "Alocar endereço IP elástico".
3. Clique em "Alocar".
4. Selecione o IP elástico alocado.
5. Clique em "Associar endereço IP elástico".
6. Em "Instância" selecione a instância desejada.
7. Clique em "Associar".

# Configuração do Banco de Dados (MySQL)

### 1. Instalar Docker e Docker Compose:
```
  sudo apt-get update
  sudo apt install docker.io -y
  sudo apt install docker-compose
```

### 2. Criar o arquivo docker compose:
* Crie o arquivo `docker-compose.yml`
```
  sudo vim docker-compose.yml
```

* Coloque as seguintes informações no arquivo `docker-compose.yml`
```yaml
version: '3.8'

services:
  mysql:
    image: mysql:8.0-oracle  # Imagem mais leve do mysql
    container_name: Mysql  # Nome do container
    environment:  # Váriaveis de ambiente mysql
      MYSQL_ROOT_PASSWORD: "teste123"  # Senha do usuário root (séra criado junto do container)
      MYSQL_USER: "root"  # Nome de usuário caso necessário
      MYSQL_PASSWORD: "teste123#"  #  Senha para o usuário
      MYSQL_DATABASE: "api"  # Nome para uma database
    ports:
      - "3306:3306"  # Porta do host e porta do container
    volumes:
      - mysql_data:/var/lib/mysql # Garante que os dados sejam persistentes

volumes:
  mysql_data: 
```

### 3. Rodar o docker-compose:
```
  sudo docker-compose up -d
```

### 4. (Opcional) Criar e atualizar os privilégios de um novo usuario e criar a database da api:
```
  sudo docker exec -it mysql-db mysql -u root -p
```
```mysql
  CREATE USER 'seu_usuario'@'%' IDENTIFIED BY 'sua_senha';
  GRANT ALL PRIVILEGES ON . TO 'seu_usuario'@'%';
  FLUSH PRIVILEGES;
  CREATE DATABASE api;
```

### 5. Atualizar as Regras de Segurança no AWS:
1. No console do Amazon EC2, vá para "Security Groups".
2. Selecione o grupo de segurança associado à instâcia do banco de dados.
3. Adicione uma regra de entrada para permitir o tráfego no porto do MySQL (3306):
* Tipo: MySQL/Aurora
* Protocolo: TCP
* Porta: 3306
* Origem: IP das máquinas do backend (use apenas ips confiáveis para maior segurança).
4. Clique em "Salvar regras".

# Configuração dos Servidores Backend

### 1. Instalar Git:
```
  sudo apt-get update
  sudo apt-get install git -y
```

### 2. Verificar instalação do Git:
```
  git --version
```

### 3. Clonar o repositório do backend:
```
  git clone https://github.com/cauancesar/Redes-Synergy.git
  cd Redes-Synergy
  git submodule init Backend
  git submodule update --remote Backend
```

### 4. Instalar Docker:
```
  sudo apt install docker.io -y
```

### 5. Configurar arquivo .env do projeto:

* No diretório do backend (Redes-Synergy/Backend/nestjs), crie um arquivo .env com as configurações necessárias e do banco de dados.
```
  sudo vim Redes-Synergy/Backend/nestjs/.env
```

Arquivo `.env`:
```.env
  DB_HOST=123.123.123.32 #ip_maquina_BD
  DB_PORT=3306           # Porta do banco de dados (normalmente 3306)
  DB_USERNAME=root       # Usuario do banco de dados Ex: root
  DB_PASSWORD=teste123    # Senha do usuario escolhido
```

### 6. Adicionar usuário ao grupo Docker e reiniciar a máquina:
```
  sudo usermod -aG docker $USER
  sudo reboot
```

### 7. Modificar o Dockerfile e construir a imagem Docker:
* Ente no diretório Backend onde o arquivo `Dockerfile` está localizado e modifique o arquivo.
```
  cd Redes-Synergy/Backend
  sudo vim Dockerfile
```

* Adicione o seguinte comando ao final do `Dockerfile`.
```dockerfile
  FROM node:20-alpine as backend
  
  # Cria o diretório do app
  WORKDIR /usr/src/app
  
  # Copia  os arquivos para a pasta de trabalho (workdir)
  COPY --chown=node:node ./nestjs .
  
  # Instala as dependencias 
  RUN npm install --quiet --no-optional --no-fund
  RUN npm run build
  
  RUN chown -R node:node /usr/src/app/dist
  
  CMD ["npm", "run", "start:dev"]  <----
  EXPOSE 5000
```

* Construa a imagem do docker.
```
  docker build -t backend .
```

### 8. Executar o container Docker:
```
  docker run -p 5000:5000 backend
```

### 9. Atualizar as Regras de Segurança no AWS:
1. No console do Amazon EC2, vá para "Security Groups".
2. Selecione o grupo de segurança associado à instâcia do backend.
3. Adicione uma regra de entrada para permitir o tráfego na porta 5000:
* Tipo: TCP personalizado
* Protocolo: TCP
* Porta: 5000
* Origem: IP da máquina do loadbalancer
4. Clique em "Salvar regras".
5. Adicione uma regra de saída para permitir o tráfego na porta 3306:
* Tipo: MYSQL/Aurora
* Protocolo: TCP
* Porta: 3306
* Origem: IP da máquina do banco de dados.
6. Clique em "Salvar regras".
  
### 10. Testar o backend:
Abra a porta 5000 no protocolo tcp para o ip 0.0.0.0 nas regras de entrada do aws para testar.
* Acesse http://ip_publico_da_maquina:5000
* Deve retornar {"message":"Unauthorized","statusCode":401} indicando que o backend está ativo, mas você não está autenticado.

### 11. Repita as etapas para as outras máquinas backend.



# Configuração do Servidor Web/Proxy
## Servindo o Frontend

### 1. Instalar Git e npm:
```
  sudo apt-get update
  sudo apt-get install git -y
  sudo apt install npm
```

### 2. Verificar instalação do Git:
```
  git --version
```

### 3. Clonar o repositório do frontend:
```
  git clone https://github.com/cauancesar/Redes-Synergy.git
  cd Redes-Synergy
  git submodule init Frontend
  git submodule update --remote Frontend
```

### 4. Configurar arquivo .env do projeto:

* No diretório do backend (Redes-Synergy/Frontend), crie um arquivo `.env` com as configurações necessárias e do banco de dados.
```
  sudo vim Redes-Synergy/Frontend/.env
```

Arquivo `.env`
```.env
  NEXTAUTH_SECRET=@teste2000
  BACKEND_URL=http://ip_loadbalancer:port    # http://backend:5000 Docker users  # local: http://localhost:5000 # http://ip_backend:port
```

### 5. Buildar e iniciar o frontend:
* No diretório raiz do frontend (Redes-Synergy/Frontend).
```
  # Caso a máquina não tenha muito poder de processamento e memória
  # Cria um arquivo de swap para expandir a memória disponível temporariamente
  sudo fallocate -l 1G /swapfile
  sudo chmod 600 /swapfile
  sudo mkswap /swapfile
  sudo swapon /swapfile

  # Execute o builde com um limite de memória
  NODE_OPTIONS="--max-old-space-size=1024" npm run build

  # Ou apenas rode
  npm run build
```
* Depois de buildar a aplicação, inicie-a.
```
  npm start
```

## Configurando o Load Balancer (usando nginx)
No arquivo `load_balancer.conf`, a diretiva `upstream backend` define um grupo de servidores que irão processar as requisições. O `proxy_pass` no bloco `location` redireciona as requisições para os servidores definidos no `upstream`.

### 1. Instalar o nginx:
```
  sudo apt update
  sudo apt install nginx
```

### 2. Criar e configurar o arquivo do load balancer:
* Criando arquivo.
```
  sudo vim /etc/nginx/conf.d/load_balancer.conf
```

* Configurações do arquivo `load_balancer.conf`.
```nginx
  upstream backend {
          server 123.123.123.213:5000;  # IP ou hostname do servidor backend
          server 123.123.123.175:5000;  # IP ou hostname do servidor backend
          server 123.123.123.160:5000;  # IP ou hostname do servidor backend
  }
  
  server {
          listen 8080;  # Porta que o load balancer irá escutar
  
          location / {
                  proxy_pass http://backend;  # Redireciona requisições para os servidores backend
          }
  }
```

### 3. Teste as configurações do nginx:
```
  sudo nginx -t
```

### 4. Reinicie o nginx para aplicar as configurações:
```
  sudo nginx -s reload
```

## Configuração do Proxy Reverso (usando nginx):

### 1. Criar e configurar o arquivo do proxy reverso:
* Com o nginx já instalado, crie o arquivo.
```
  sudo vim /etc/nginx/sites-available/proxy_reverso.conf
```

* Configurações do arquivo `proxy_reverso.conf`.
```nginx
  server {
      listen 10.0.0.1:80;   # Limita o proxy reverso para o ip 10.0.0.1 na porta 80
      server_name 10.0.0.1; # IP do servidor da vpn | ou ip publico da máquina do proxy reverso
  
  
      location / {
          proxy_pass http://ip_da_maquina_frontend:3000;  # Redireciona para o servidor Next.js em execução na porta 3000
          proxy_set_header Host $host;
          proxy_set_header X-Real-IP $remote_addr;
          proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
  
          proxy_connect_timeout 5s;
          proxy_send_timeout 10s;     # Tempo máximo para enviar dados
          proxy_read_timeout 10s;     # Tempo máximo para receber dados
          proxy_next_upstream error timeout http_502 http_503 http_504;
  
      }
  
      location /api/ { # Necessário para uma rota de login específica da aplicação Synergy
          proxy_pass http://ip_da_maquina_frontend:3000;  # Redireciona para o servidor Next.js em execução na porta 3000
          proxy_set_header Host $host;
          proxy_set_header X-Real-IP $remote_addr;
          proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
      }
  
      location /synergy/ {
          proxy_pass http://ip_da_maquina_loadbalancer:8080/;  # IP e porta do Load Balancer | com a "/" no final do ip o nginx remove o /synergy/ antes de repassar para o loadbalancer
      }
  }
```

### 3. Linkar o arquivo de configuração ao arquivo em uso:
```
  sudo ln -s /etc/nginx/sites-available/proxy_reverso.conf /etc/nginx/sites-enabled/
```

### 4. Teste as configurações do nginx:
```
  sudo nginx -t
```

### 5. Reinicie o nginx para aplicar as configurações:
```
  sudo nginx -s reload
```

## Configuração do VPN (usando OpenVPN):
O OpenVPN permite a criação de redes privadas seguras. O servidor `server.conf` define os parâmetros da conexão, enquanto o cliente `client.conf` contém as configurações para se conectar ao servidor.

### 1. Instalar o OpenVPN:
```
  sudo apt update
  sudo apt install openvpn -y
```

### 2. Entre na pasta openvpn e exclua as pastas client e server:
```
  cd /etc/openvpn/
  sudo rm -r client/
  sudo rm -r server/
```

### 3. Crie e configure o arquivo do servidor:
* Faça login como usuário root
```
 sudo su
```

* Criando o arquivo `server.conf`
```
  vim server.conf
```

* Configurando o arquivo `server.conf`
```conf
  dev tun
  ifconfig 10.0.0.1 10.0.0.2 # IP do servidor / IP do cliente
  secret /caminho/para/chave
  port 1194 # Porta do openvpn (pode ser qualquer outra)
  proto udp
  comp-lzo
  verb 4
  keepalive 10 120
  persist-key
  persist-tun
  float
  cipher AES256
```

### 4. Crie e configure o arquivo do cliente:
* Com o arquivo `server.conf` criado, faça uma copia de `server.conf` para o arquivo `client.conf`
```
  cat server.conf > client.conf
```

* Edite o arquivo `client.conf`
```conf
  dev tun
  ifconfig 10.0.0.2 10.0.0.1 # IP do cliente / IP do servidor
  remote ip_publico_da_maquina_vpn
  secret /caminho/para/chave
  port 1194 # Porta do openvpn (pode ser qualquer outra)
  proto udp
  comp-lzo
  verb 4
  keepalive 10 120
  persist-key
  persist-tun
  float
  cipher AES256
```

### 5. Crie a chave do openvpn copie para o usuário linux logado e dê as permissões necessárias:
* Criando a chave.
```
  openvpn --genkey --secret chave
```

* Copiando a chave para o usuário ubuntu.
```
  cp chave /home/ubuntu
```

* Dê a permissão para o usuário ubuntu.
```
  chown ubuntu:ubuntu /home/ubuntu/chave
  ll /home/ubuntu/chave  # Para verificar se as permissões foram bem sucedidas.
```

* Copie o arquivo client.conf para o usuario ubuntu e dê as mesmas permissões.
```
  cp client.conf /home/ubuntu
  chown ubuntu:ubuntu /home/ubuntu/client.conf
  ll /home/ubuntu/client.conf  # Para verificar se as permissões foram bem sucedidas.
```
### 6. Faça o download da chave e do arquivo client.conf para a máquina do cliente:
```                                                                 
  scp -i <Aquivo .pem> ubuntu@123.123.123.123:/caminho/para/chave .
  scp -i <Aquivo .pem> ubuntu@123.123.123.123:/caminho/para/cliente.conf .
```

### 7. Inicie o servidor do VPN:
```
  openvpn --config server.conf
```

## Atualizar as Regras de Segurança no AWS:

### Configuração da VPN

1 - No console do Amazon EC2, vá para "Security Groups".

2 - Selecione o grupo de segurança associado à instância que atua como servidor VPN.

3 - Adicione uma regra de entrada para permitir o tráfego na porta 1194 (utilizada pela VPN):
  * Tipo: UDP personalizado
  * Protocolo: UDP
  * Porta: 1194
  * Origem: 0.0.0.0/0 (para acesso de qualquer endereço IP).

4 - Clique em "Salvar regras" para aplicar a nova regra.

### Configuração do Frontend

1 - No console do Amazon EC2, vá para "Security Groups".

2 - Selecione o grupo de segurança associado à instância do Frontend.

3 - Adicione uma regra de entrada para permitir o tráfego na porta 3000 (utilizada pelo Frontend):
  * Tipo: TCP personalizado
  * Protocolo: TCP
  * Porta: 3000
  * Origem: ip_publico_maquina_Web/Proxy (substitua ip_publico_maquina_Web/Proxy pelo IP público da instância Web/Proxy).

4 - Clique em "Salvar regras" para aplicar a nova regra.

### Configuração do Load Balancer

1 - No console do Amazon EC2, vá para "Security Groups".

2 - Selecione o grupo de segurança associado à instância do Load Balancer.

3 - Adicione uma regra de entrada para permitir o tráfego na porta 8080 (utilizada pelo Load Balancer):
  * Tipo: TCP personalizado
  * Protocolo: TCP
  * Porta: 8080
  * Origem: ip_publico_maquina_Web/Proxy (substitua ip_publico_maquina_Web/Proxy pelo IP público da instância Web/Proxy).

4 - Clique em "Salvar regras" para aplicar a nova regra.

## Como se conectar ao servidor VPN usando o OpenVPN GUI no Windows:
### 1. Baixe e instale o OpenVPN GUI no seu computador Windows.

### 2. Copie os arquivos client.conf e chave que você baixou anteriormente para a pasta de configuração do OpenVPN:
```
  C:\Program Files\OpenVPN\config
```

### 3. Abra o OpenVPN GUI como Administrador (clique com o botão direito do mouse no ícone e selecione "Executar como administrador").

### 4. Você verá o ícone do OpenVPN na bandeja do sistema (canto inferior direito da tela). Clique com o botão direito do mouse no ícone.

### 5. Modifique o arquivo client.conf:
```ovpn
  dev tun
  ifconfig 10.0.0.2 10.0.0.1
  remote ip_publico_maquina_VPN
  port 1194
  secret C:\\caminho\\para\\chave
  proto udp
  verb 4
  keepalive 10 120
  persist-key
  persist-tun
  float
  cipher AES256
  auth SHA256
```
> Salve o arquivo como .ovpn (para funcionar no windows).

### 6. No menu, selecione a opção correspondente ao seu perfil de VPN (que deve ser o nome do arquivo client.conf).

### 7. Clique em "Conectar". O OpenVPN tentará estabelecer a conexão com o servidor VPN. Você verá mensagens de status na janela do OpenVPN GUI.

### 8. Se a conexão for bem-sucedida, você verá uma mensagem de confirmação. Agora você estará conectado à VPN!

# (Opcional) - Adicionar certificação ao OpenVPN usando EasyRSA

### 1. Instalar o EasyRSA:
```
  sudo su
  apt update
  apt install easy-rsa
```

### 2. Copiar a pasta do EasyRSA para o diretório do OpenVPN
Copie a pasta do EasyRSA para dentro do diretório /etc/openvpn:
```
  cd /etc/openvpn/
  cp -r /usr/share/easy-rsa .
```

### 3. Criar e editar o arquivo vars dentro da pasta EasyRSA
Entre no diretório EasyRSA e crie o arquivo `vars`:
```
  cd easy-rsa/
  vim vars
```

`vars`
```conf
if [ -z "$EASYRSA_CALLER" ]; then
        echo "You appear to be sourcing an Easy-RSA *vars* file. This is" >&2
        echo "no longer necessary and is disallowed. See the section called" >&2
        echo "*How to use this file* near the top comments for more details." >&2
        return 1
fi

set_var EASYRSA "${0%/*}"
set_var EASYRSA_PKI             "$PWD/pki"
set_var EASYRSA_TEMP_DIR        "$EASYRSA_PKI"
set_var EASYRSA_DN      "cn_only"
set_var EASYRSA_KEY_SIZE        2048
set_var EASYRSA_CA_EXPIRE       3650
set_var EASYRSA_CERT_EXPIRE     825
```

### 4. Execute os comandos para criar a infraestrutura de certificados
Execute os seguintes comandos para inicializar o PKI (Infraestrutura de Chaves Públicas) e gerar os certificados:
```
# Inicializa o diretório PKI (Public Key Infrastructure) onde os certificados serão armazenados
./easyrsa init-pki

# Cria uma autoridade certificadora (CA) raiz, que será usada para assinar os certificados dos servidores e clientes
./easyrsa build-ca

# Gera a solicitação de assinatura de certificado (CSR) para o servidor, sem senha (nopass)
# Isso cria a chave privada e o certificado de requisição do servidor
./easyrsa gen-req web nopass

# Assina o certificado de requisição do servidor (web) com a CA que foi criada na etapa anterior
# Isso gera o certificado final para o servidor
./easyrsa sign-req server web

# Gera a solicitação de assinatura de certificado (CSR) para o cliente, sem senha (nopass)
# Isso cria a chave privada e o certificado de requisição do cliente
./easyrsa gen-req client nopass

# Assina o certificado de requisição do cliente (client) com a CA que foi criada na etapa anterior
# Isso gera o certificado final para o cliente
./easyrsa sign-req client client

# Gera os parâmetros Diffie-Hellman (DH), usados para a troca segura de chaves durante a conexão VPN
# Esse arquivo é necessário para a configuração do servidor OpenVPN
./easyrsa gen-dh
```
* `init-pki`: Prepara o ambiente para criar e armazenar certificados.
* `build-ca`: Cria a autoridade certificadora (CA) que será usada para assinar os certificados do servidor e cliente.
* `gen-req`: Gera uma chave privada e um arquivo de requisição de certificado (CSR), que é usado para pedir a assinatura do certificado pela CA.
* `sign-req`: Assina o CSR com a CA para criar o certificado final.
* `gen-dh`: Gera parâmetros de Diffie-Hellman para troca de chaves seguras durante a conexão VPN.

### 5. Criar a chave TLS para autenticação
Gere a chave TLS para proteger a comunicação entre o servidor e os clientes:
```
cd /etc/openvpn
openvpn --genkey secret ta.key
```

### 6. Atualizar o arquivo server.conf e o arquivo client
Arquivo `server.conf`:
```conf
dev tun
port 1194 # Porta do openvpn (pode ser qualquer outra)
proto udp

ifconfig 10.0.0.1 10.0.0.2
push "ifconfig 10.0.0.2 10.0.0.1"
push "route 10.0.0.0 255.255.255.0"

ca /caminho/para/ca.crt
cert /caminho/para/web.crt
key /caminho/para/web.key
dh /caminho/para/dh.pem
tls-auth /caminho/para/ta.key 0

tls-server
cipher AES-256-GCM
auth SHA256

comp-lzo
verb 4

keepalive 10 120
persist-key
persist-tun
float
```

Arquivo `client.ovpn` (configuração do cliente para Windows):
```conf
client
dev tun
proto udp
remote ip_publico_da_maquina_vpn 1194
resolv-retry infinite
nobind
comp-lzo
persist-key
persist-tun
remote-cert-tls server
cipher AES-256-GCM
auth SHA256
key-direction 1
verb 3
disable-dco


ca C:/caminho/para/ca.crt
cert C:/caminho/para/client.crt
key C:/caminho/para/client.key
tls-auth C:/caminho/para/ta.key
```

### 7. Copiar os arquivos necessários para o cliente:
```
cd /etc/openvpn
mkdir /home/ubuntu/client  # Cria uma pasta chamada client
cp easy-rsa/pki/issued/client.crt /home/ubuntu/client/
cp easy-rsa/pki/private/client.key /home/ubuntu/client/
cp ta.key  /home/ubuntu/client/
cp easy-rsa/pki/ca.crt /home/ubuntu/client/
```

### 8. Definir permissões para os arquivos

Defina as permissões apropriadas para os arquivos copiados:
```
cd /home/ubuntu/client
chown ubuntu:ubuntu *
```

### 9. Transferir os arquivos para a máquina cliente
Use o comando scp para transferir os arquivos necessários para a máquina cliente:
```
  scp -i <Aquivo .pem> ubuntu@123.123.123.123:/caminho/para/client/* .
```

### 10. Iniciar o servidor OpenVPN e testar a conexão:
```
  killall openvpn
  openvpn --config /caminho/para/server.conf
```

# (Opcional) - Adicionar Health Check ao Load Balancer no Nginx

### 1. Instalar as dependências necessárias:
```
sudo su
apt update
apt install -y build-essential libpcre3 libpcre3-dev zlib1g zlib1g-dev libssl-dev
```

### 2. Baixar o código-fonte do nginx
Baixe e extraia o código-fonte do nginx:
```
cd /usr/local/src
wget http://nginx.org/download/nginx-1.24.0.tar.gz
tar -xzvf nginx-1.24.0.tar.gz
cd nginx-1.24.0
```

### 3. Baixar o módulo nginx-upstream-check-module
Clone o repositório do módulo de verificação de saúde:
```
git clone https://github.com/yaoweibin/nginx_upstream_check_module.git
```

### 4. Aplicar o patch no código do Nginx:
```
patch -p1 < /caminho/para/nginx_upstream_check_module/check_1.20.1+.patch
```

### 5. Configurar a compilação:
```
./configure \
--prefix=/usr/share/nginx \
--sbin-path=/usr/sbin/nginx \
--modules-path=/usr/lib/nginx/modules \
--conf-path=/etc/nginx/nginx.conf \
--error-log-path=/var/log/nginx/error.log \
--http-log-path=/var/log/nginx/access.log \
--pid-path=/run/nginx.pid \
--lock-path=/var/lock/nginx.lock \
--http-client-body-temp-path=/var/lib/nginx/body \
--http-proxy-temp-path=/var/lib/nginx/proxy \
--http-fastcgi-temp-path=/var/lib/nginx/fastcgi \
--with-http_ssl_module \
--with-stream \
--with-http_v2_module \
--add-module=./nginx_upstream_check_module
```

### 6. Compilar e instalar o nginx:
```
make -j$(nproc)
make install
```
> O parâmetro -j$(nproc) otimiza o processo de compilação utilizando todos os núcleos do processador disponíveis.

### 7. Verificar a versão e os módulos instalados
Após a instalação, verifique se o módulo foi corretamente instalado executando:
```
nginx -V
```
Procure pela linha nginx_upstream_check_module na saída para confirmar que o módulo foi carregado corretamente.

### 8. Modificar o arquivo load_balancer.conf
No arquivo de configuração do nginx, adicione a configuração de health checks.
```conf
  upstream backend {
          server ip_publico_backend1:5000;  # IP ou hostname do servidor backend
          server ip_publico_backend2:5000;  # IP ou hostname do servidor backend
          server ip_publico_backend3:5000;  # IP ou hostname do servidor backend

          check interval=5000 rise=1 fall=3 timeout=4000; <--- Adicione essa linha

          # interval=5000: intervalo entre os verificações de saúde (5 segundos)
          # rise=1: número de tentativas bem-sucedidas necessárias para considerar o servidor saudável
          # fall=3: número de falhas consecutivas necessárias para considerar o servidor inativo
          # timeout=4000: tempo máximo para a resposta de cada verificação de saúde (4 segundos)
  }

  server {
          listen 8080;  # Porta que o load balancer irá escutar

          location / {
                proxy_pass http://backend;  # Redireciona requisições para os servidores backend
          }
  }
```

### 9. Testar e reiniciar o nginx
Depois de modificar o arquivo de configuração, teste a configuração do Nginx e reinicie o serviço para aplicar as mudanças:
```
nginx -t
systemctl restart nginx
```