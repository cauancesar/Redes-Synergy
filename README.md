# Redes-Synergy
Configurações necessárias para implementar uma arquitetura de rede usando máquinas da aws.

<img src="https://github.com/cauancesar/Redes-Synergy/blob/main/imgs/_TopologiaRedes.drawio.png" height=800></img>

## Configuração da AWS

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

## Configuração do Banco de Dados (MySQL)

### 1. *Instalar o MySQL*:
```
  sudo apt-get update
  sudo apt-get install mysql-server -y
```

### 2. *Iniciar e habilitar o MySQL*:
```
  sudo systemctl enable --now mysql
```

### 3. *Verificar status do MySQL*:
```
  sudo systemctl status mysql
```

### 4. *Alterar IP de escuta para 0.0.0.0*:
```
  sudo vim /etc/mysql/mysql.conf.d/mysqld.cnf
  # Alterar a linha bind-address para 0.0.0.0
```

### 5. *Reiniciar o MySQL para aplicar as mudanças*:
```
  sudo systemctl restart mysql
```

### 6. *Opcional - Liberar a porta 3306 (caso esteja usando firewall)*:
```
  sudo ufw allow 3306/tcp
  sudo ufw status
```

### 7. *Atualizar as Regras de Segurança no AWS*:
1. No console do Amazon EC2, vá para "Security Groups".
2. Selecione o grupo de segurança associado à instâcia do banco de dados.
3. Adicione uma regra de entrada para permitir o tráfego no porto do MySQL (3306):
* Tipo: MySQL/Aurora
* Protocolo: TCP
* Porta: 3306
* Origem: IP das máquinas do backend
4. Clique em "Salvar regras".

<hr/>

## Configuração dos Servidores Backend

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

### 5. *Configurar Docker no projeto*:

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
```
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
