# Redes-Synergy
Configurações necessárias para implementar uma arquitetura de rede usando máquinas da aws.

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
```

### 3. *Rodar o docker-compose*:
```
  sudo docker-compose up -d
```

### 4. *Opcional - Liberar a porta 3306 (caso esteja usando firewall)*:
```
  sudo ufw allow 3306/tcp
  sudo ufw status
```

### 5. *Atualizar as Regras de Segurança no AWS*:
1. No console do Amazon EC2, vá para "Security Groups".
2. Selecione o grupo de segurança associado à instâcia do banco de dados.
3. Adicione uma regra de entrada para permitir o tráfego no porto do MySQL (3306):
* Tipo: MySQL/Aurora
* Protocolo: TCP
* Porta: 3306
* Origem: IP das máquinas do backend
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
  sudo vim Redes-Synergy\Backend\nestjs\.env
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
  cd Redes-Synergy\Backend
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

### 1. *Instalar Git*:
```
  sudo apt-get update
  sudo apt-get install git -y
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
  sudo vim Redes-Synergy\Backend\nestjs\.env
```
```.env
  NEXTAUTH_SECRET=@teste2000
  BACKEND_URL=http://ip_backend:port(docker)    # http://backend:5000 Docker users  # local: http://localhost:5000 # http://ip_backend:port
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
      server_name 52.22.78.70;
  
  
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

## *Configurando a VPN*:
