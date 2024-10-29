# Redes-Synergy
Configurações necessárias para implementar uma arquitetura de rede usando máquinas da aws.

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

### 9. *Testar o backend*:
* Acesse http://ip_publico_da_maquina:5000
* Deve retornar {"message":"Unauthorized","statusCode":401} indicando que o backend está ativo, mas você não está autenticado.

### 10. *Repita as etapas para as outras máquinas backend.*

<hr/>
