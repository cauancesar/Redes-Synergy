# Redes-Synergy
## Introdução
Este projeto implementa uma arquitetura de rede escalável usando instâncias da AWS. O objetivo é criar um ambiente robusto que suporte aplicações modernas, utilizando Docker para a contêinerização e Nginx para gerenciamento de tráfego. Ao seguir este guia, você aprenderá a configurar servidores backend, um banco de dados MySQL e um servidor web com balanceamento de carga.


<img src="https://github.com/cauancesar/Redes-Synergy/blob/main/imgs/_TopologiaRedes.drawio.png" height=800></img>

## *Configuração da AWS*

### 1. *Criar uma instância EC2 para cada máquina*:
1. Acesse o console do Amazon EC2.
2. Clique em "Executar Instância".
3. De um nome a instância.
4. Em "Imagens de aplicação e de sistema operacional (imagem de máquina da Amazon)" clique em Ubuntu.
5. Em "Par de chaves (login)" crie e selecione um novo par de chaves para acessar a máquina.
6. Em "Configurações de rede" selecione as opções "Permitir tráfego HTTP da Internet" e "Permitir tráfego SSH de".
7. *Opcional* - Em "Configurar armazenamento" aumento o tamanho do armazenamento caso necessário.
8. Clique em "Executar Instância".

### 2. *Alocar um IP elástico às instâncias*:
1. No console do Amazon EC2, no painel de navegação, clique em “IPs elásticos”.
2. Clique em "Alocar endereço IP elástico".
3. Clique em "Alocar".
4. Selecione o IP elástico alocado.
5. Clique em "Associar endereço IP elástico".
6. Em "Instância" selecione a instância desejada.
7. Clique em "Associar".

## *Configuração do Banco de Dados (MySQL)*

### 1. *Instalar Docker e Docker Compose*:
```
  sudo apt-get update
  sudo apt install docker.io -y
  sudo apt  install docker-compose
```

### 2. *Criar o arquivo docker compose*:
* Crie o arquivo docker-compose.yml
```
  sudo vim docker-compose.yml
```

* Coloque as seguintes informações
```yaml
version: '3.8'

services:
  mysql:
    image: mysql:8.0-oracle  # Imagem mais leve do mysql
    container_name: Mysql  # Nome do container
    environment:  # Váriaveis de ambiente mysql
      MYSQL_ROOT_PASSWORD: "teste123"  # Senha do usuário root (séra criado junto do container)
      MYSQL_USER: "Syatt"  # Nome de usuário caso necessário
      MYSQL_PASSWORD: "Senha123#"  #  Senha para o usuário
      MYSQL_DATABASE: "api"  # Nome para uma database
    ports:
      - "3306:3306"  # Porta do host e porta do container
    volumes:
      - mysql_data:/var/lib/mysql # Garante que os dados sejam persistentes

volumes:
  mysql_data: 
```

### 3. *Rodar o docker-compose*:
```
  sudo docker-compose up -d
```

### 4. *Opcional - Criar e atualizar os privilégios de um novo usuario e criar a database da api*:
```
  sudo docker exec -it mysql-db mysql -u root -p
```
```mysql
  CREATE USER 'seu_usuario'@'%' IDENTIFIED BY 'sua_senha';
  GRANT ALL PRIVILEGES ON . TO 'seu_usuario'@'%';
  FLUSH PRIVILEGES;
  CREATE DATABASE api;
```

### 5. *Atualizar as Regras de Segurança no AWS*:
1. No console do Amazon EC2, vá para "Security Groups".
2. Selecione o grupo de segurança associado à instâcia do banco de dados.
3. Adicione uma regra de entrada para permitir o tráfego no porto do MySQL (3306):
* Tipo: MySQL/Aurora
* Protocolo: TCP
* Porta: 3306
* Origem: IP das máquinas do backend (use apenas ips confiáveis para maior segurança).
4. Clique em "Salvar regras".

<hr/>

## *Configuração dos Servidores Backend*

### 1. *Instalar Git*:
```
  sudo apt-get update
  sudo apt-get install git -y
```

### 2. *Verificar instalação do Git*:
```
  git --version
```

### 3. *Clonar o repositório do backend*:
```
  git clone https://github.com/cauancesar/Redes-Synergy.git
  cd Redes-Synergy
  git submodule init Backend
  git submodule update --remote Backend
```

### 4. *Instalar Docker*:
```
  sudo apt install docker.io -y
```

### 5. *Configurar arquivo .env do projeto*:

* No diretório do backend (Redes-Synergy/Backend/nestjs), crie um arquivo .env com as configurações necessárias e do banco de dados.
```
  sudo vim Redes-Synergy/Backend/nestjs/.env
```
```.env
  DB_HOST=123.123.123.32 #ip_maquina_BD
  DB_PORT=3306           # Porta do banco de dados (normalmente 3306)
  DB_USERNAME=root       # Usuario do banco de dados Ex: root
  DB_PASSWORD=exemplo    # Senha do usuario escolhido
```

### 6. *Adicionar usuário ao grupo Docker e reiniciar a máquina*:
```
  sudo usermod -aG docker $USER
  sudo reboot
```

### 7. *Modificar o Dockerfile e construir a imagem Docker*:
* Ente no diretório Backend onde o Dockerfile está localizado e modifique o arquivo.
```
  cd Redes-Synergy/Backend
  sudo vim Dockerfile
```

* Adicione ao final do Dockerfile.
```dockerfile
    CMD ["npm", "run", "start:dev"]
```

* Construa a imagem do docker.
```
  docker build -t backend .
```

### 8. *Executar o container Docker*:
```
  docker run -p 5000:5000 backend
```

### 9. *Atualizar as Regras de Segurança no AWS*:
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
* Origem: IP da máquina do banco de dados ou selecione o grupo de segurança da instância associada ao banco de dados.
6. Clique em "Salvar regras".
  
### 10. *Testar o backend*:
* Acesse http://ip_publico_da_maquina:5000
* Deve retornar {"message":"Unauthorized","statusCode":401} indicando que o backend está ativo, mas você não está autenticado.

### 11. *Repita as etapas para as outras máquinas backend.*

<hr/>

## Configuração do Servidor Web/Proxy
## *Servindo o Frontend*

### 1. *Instalar Git e npm*:
```
  sudo apt-get update
  sudo apt-get install git -y
  sudo apt install npm
```

### 2. *Verificar instalação do Git*:
```
  git --version
```

### 3. *Clonar o repositório do frontend*:
```
  git clone https://github.com/cauancesar/Redes-Synergy.git
  cd Redes-Synergy
  git submodule init Frontend
  git submodule update --remote Frontend
```

### 4. *Configurar arquivo .env do projeto*:

* No diretório do backend (Redes-Synergy/Backend/nestjs), crie um arquivo .env com as configurações necessárias e do banco de dados.
```
  sudo vim Redes-Synergy/Backend/nestjs/.env
```
```.env
  NEXTAUTH_SECRET=@teste2000
  BACKEND_URL=http://ip_loadbalancer:port    # http://backend:5000 Docker users  # local: http://localhost:5000 # http://ip_backend:port
```

### 5. *Buildar e iniciar o frontend*:
* No diretório raiz do frontend (Redes-Synergy/Frontend).
```
  export NODE_OPTIONS="--max-old-space-size=512"  \\ Caso a máquina não tenha muito espaço de memória disponível.
  npm run build
```
* Depois de buildar a aplicação, inicie-a.
```
  npm start
```

## *Configurando o Load Balancer (usando nginx)*

### 1. *Instalar o nginx*:
```
  sudo apt update
  sudo apt install nginx
```

### 2. *Criar e configurar o arquivo do load balancer*:
* Criando arquivo.
```
  sudo vim /etc/nginx/conf.d/load_balancer.conf
```
* Configurações do arquivo load_balancer.conf.
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

### 3. *Teste as configurações do nginx*:
```
  sudo nginx -t
```

### 4. *Reinicie o nginx para aplicar as configurações*:
```
  sudo nginx -s reload
```

## *Configurando o Proxy Reverso (usando nginx)*:

### 1. *Criar e configurar o arquivo do proxy reverso*:
* Com o nginx já instalado, crie o arquivo.
```
  sudo vim /etc/nginx/sites-available/proxy_reverso.conf
```

* Configurações do arquivo proxy_reverso.conf.
```nginx
  server {
      listen 80;
      server_name ip_da_maquina_proxy_reverso;
  
  
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

### 3. *Linkar o arquivo de configuração ao arquivo em uso*:
```
  sudo ln -s /etc/nginx/sites-available/proxy_reverso.conf /etc/nginx/sites-enabled/
```

### 4. *Teste as configurações do nginx*:
```
  sudo nginx -t
```

### 5. *Reinicie o nginx para aplicar as configurações*:
```
  sudo nginx -s reload
```

## *Configurando a VPN (usando OpenVPN)*:

### 1. *Instalar o OpenVPN*:
```
  sudo apt update
  sudo apt install openvpn -y
```

### 2. *Entre na pasta openvpn e exclua as pastas client e server*:
```
  cd /etc/openvpn/
  rm -r client/
  rm -r server/
```

### 3. *Crie e configure o arquivo do servidor*:
* Faça login como usuário root
```
 sudo su
```

* Criando o arquivo server.conf
```
  vim server.conf
```

* Configurando o arquivo server.conf
```conf
  dev tun
  ifconfig 10.0.0.1 10.0.0.2 # IP privado da máquina | IP do servidor / IP do cliente
  secret /etc/openvpn/chave
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

### 4. *Crie e configure o arquivo do cliente*:
* Com o arquivo server.conf criado, faça uma copia de server.conf para o arquivo client.conf
```
  cat server.conf > client.conf
```

* Edite o arquivo client.conf
```conf
  dev tun
  ifconfig 10.0.0.2 10.0.0.1 # IP privado da máquina | IP do cliente / IP do servidor
  remote ip_publico_da_maquina
  secret /etc/openvpn/chave
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

### 5. *Crie a chave do openvpn copie para o usuário linux logado e dê as permissões necessárias*:
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
  ll /home/ubuntu/chave  # Para verificar se as permissões foram bem sucedidas.
```
### 6. *Ative o encaminhamento de IP*:
* Esses comandos ativam o encaminhamento de pacotes IP, que é necessário para que os clientes da VPN possam se comunicar com a rede externa.
```
  echo 1 > /proc/sys/net/ipv4/ip_forward
  sysctl -p  # Aplica as alterações de configuração do sistema.
```

### 7. *Configure as regras do iptables*:
* Essas regras permitem que o tráfego da rede privada 10.0.0.0/24 seja roteado corretamente através do servidor VPN e redirecionado para a interface externa.
```
  iptables -t nat -A POSTROUTING -s 10.0.0.0/24 -o eth0 -j MASQUERADE
  iptables -A FORWARD -i tun0 -o eth0 -j ACCEPT
  iptables -A FORWARD -i eth0 -o tun0 -j ACCEPT
  iptables-save  # Salva as regras do iptables para que elas persistam após reinicializações.
```

### 8. *Faça o download da chave e do arquivo client.conf para a máquina do cliente*:
```                                                                 
                                                                (usuário do par de chaves e ip publico da maquina)
  scp -i <Aquivo .pem/Par de chaves criado para as instâncias da AWS> ubuntu@123.123.123.123:/caminho-para-o-arquivo/home/ubuntu/chave .
  scp -i <Aquivo .pem/Par de chaves criado para as instâncias da AWS> ubuntu@123.123.123.123:/caminho-para-o-arquivo/home/ubuntu/cliente.conf .
```

### 9. *Inicie o servidor do VPN*:
```
  openvpn --config server.conf
```

### 9. *Atualizar as Regras de Segurança no AWS*:

#### *Configuração da VPN*

1 - No console do Amazon EC2, vá para "Security Groups".

2 - Selecione o grupo de segurança associado à instância que atua como servidor VPN.

3 - Adicione uma regra de entrada para permitir o tráfego na porta 1194 (utilizada pela VPN):
  * Tipo: UDP personalizado
  * Protocolo: UDP
  * Porta: 1194
  * Origem: 0.0.0.0/0 (para acesso de qualquer endereço IP).

4 - Clique em "Salvar regras" para aplicar a nova regra.

#### *Configuração do Frontend*

1 - No console do Amazon EC2, vá para "Security Groups".

2 - Selecione o grupo de segurança associado à instância do Frontend.

3 - Adicione uma regra de entrada para permitir o tráfego na porta 3000 (utilizada pelo Frontend):
  * Tipo: TCP personalizado
  * Protocolo: TCP
  * Porta: 3000
  * Origem: ip_publico_maquina_Web/Proxy (substitua ip_publico_maquina_Web/Proxy pelo IP público da instância Web/Proxy).

4 - Clique em "Salvar regras" para aplicar a nova regra.

#### *Configuração do Load Balancer*

1 - No console do Amazon EC2, vá para "Security Groups".

2 - Selecione o grupo de segurança associado à instância do Load Balancer.

3 - Adicione uma regra de entrada para permitir o tráfego na porta 8080 (utilizada pelo Load Balancer):
  * Tipo: TCP personalizado
  * Protocolo: TCP
  * Porta: 8080
  * Origem: ip_publico_maquina_Web/Proxy (substitua ip_publico_maquina_Web/Proxy pelo IP público da instância Web/Proxy).

4 - Clique em "Salvar regras" para aplicar a nova regra.

### 10. *Como se conectar ao servidor VPN usando o OpenVPN GUI no Windows*:
1 - Baixe e instale o OpenVPN GUI no seu computador Windows.

2 - Copie os arquivos client.conf e chave que você baixou anteriormente para a pasta de configuração do OpenVPN:
```
  C:\Program Files\OpenVPN\config
```

3 - Abra o OpenVPN GUI como Administrador (clique com o botão direito do mouse no ícone e selecione "Executar como administrador").

4 - Você verá o ícone do OpenVPN na bandeja do sistema (canto inferior direito da tela). Clique com o botão direito do mouse no ícone.

5 - Modifique o arquivo client.conf:
```ovpn
  dev tun
  ifconfig 10.0.0.2 10.0.0.1
  remote ip_publico_maquina_VPN
  port 1194
  secret C:\\Caminho-Para-A-Chave\\chave
  proto udp
  verb 4
  keepalive 10 120
  persist-key
  persist-tun
  float
  cipher AES256
  auth SHA256
```
  * Salve o arquivo como .ovpn (para funcionar no windows).

6 - No menu, selecione a opção correspondente ao seu perfil de VPN (que deve ser o nome do arquivo client.conf).

7 - Clique em "Conectar". O OpenVPN tentará estabelecer a conexão com o servidor VPN. Você verá mensagens de status na janela do OpenVPN GUI.

8 - Se a conexão for bem-sucedida, você verá uma mensagem de confirmação. Agora você estará conectado à VPN!

