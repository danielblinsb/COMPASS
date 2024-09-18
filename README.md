
### COMPASS - PB - SENAC/UNICESUMAR

#### Atividade de Linux / Prática

---

### REQUISITOS AWS

1. **Chave Pública para Acesso ao Ambiente:**
   - Serviço EC2 > Network & Security > Key Pairs > Create Key Pair
   - Tipo de chave para uso com Open SSH: escolha ".pem"
   - Criar chave, o arquivo de chave privada é baixado automaticamente. Guardar em segurança para uso posterior no acesso à instância através de Open SSH.

2. **Instância EC2:**
   - Serviço EC2 > Instances > Launch instances
   - Name and tags: configurar de acordo com o projeto
   - Application and OS Images: "Amazon Linux 2 AMI (HVM) - Kernel 5.10, SSD Volume Type"
   - Tipo da instância: "t3.small"
   - Key pair: selecionar chave criada anteriormente
   - Network settings: padrão da console (não modificar)
   - Configure Storage: selecionar 16GB gp3
   - Launch instance

3. **Elastic IP:**
   - Serviço EC2 > EC2 Dashboard > Network & Security > Elastic Ips > Allocate Elastic IP address
   - Elastic IP address settings: padrão da console (não modificar)
   - Tags: configurar de acordo com o projeto
   - Allocate
   - Selecionar o endereço IP alocado > Actions > Associate Elastic IP address
   - Resource type: Instance
   - Escolher instância criada anteriormente > Associate

4. **Liberando Portas de Comunicação para Acesso Público:**
   - Serviço EC2 > Instances > Network & Security > Security Groups > Selecionar SG do projeto
   - Inbound rules > Edit inbound rules > Add rule
     - Type: Custom TCP > Protocol: TCP > Port range: 111  > Source: Anywhere-IPv4
     - Type: Custom UDP > Protocol: UDP > Port range: 111  > Source: Anywhere-IPv4
     - Type: Custom TCP > Protocol: TCP > Port range: 2049 > Source: Anywhere-IPv4
     - Type: Custom UDP > Protocol: UDP > Port range: 2049 > Source: Anywhere-IPv4
     - Type: Custom TCP > Protocol: TCP > Port range: 80   > Source: Anywhere-IPv4
     - Type: Custom TCP > Protocol: TCP > Port range: 443  > Source: Anywhere-IPv4
   - Save rules

---

### REQUISITOS NO LINUX

5. **Configurar o NFS (Network File System):**
   - Usando uma VM com sistema Linux, acessar a instância via OpenSSH
   - Copiar a chave ".pem" para a máquina virtual
   - Atribuir permissões de leitura à chave somente para o proprietário do arquivo com o comando `chmod 400 path-to-your-key-pair.pem`
   - Conectar à instância com o comando `ssh -i caminho/para/sua-chave.pem ec2-user@seu-endereco-ip`
   - Verificar distribuição Linux usada na instância com `cat /etc/os-release`
   - Comando para o NFS: `sudo yum install nfs-utils` (já instalado na última versão)
   - Verificar status do serviço: `sudo systemctl status nfs-server`
   - Iniciar o serviço NFS: `sudo systemctl start nfs-server`

6. **Criar um Diretório dentro do Filesystem do NFS com seu nome:**
   - `sudo mkdir /mnt/nfs_share` (substituir nfs_share por seu nome)
   - Configurar permissões no diretório compartilhado para todos: `sudo chmod 777 /mnt/seu-nome`
   - Editar arquivo de exportação: `sudo nano /etc/exports`
   - Adicionar linha: `/mnt/seu-nome *(rw,sync,no_root_squash,insecure)`
   - Salvar arquivo e aplicar: `sudo exportfs -ra`
   - Reiniciar o serviço NFS: `sudo systemctl restart nfs-server.service`

   - No cliente (VM com sistema Linux Debian), instalar o cliente NFS: `sudo apt install nfs-common`
   - Criar ponto de montagem: `sudo mkdir /mnt/seu-nome`
   - Montar diretório compartilhado: `sudo mount -t nfs IP-da-instancia:/mnt/seu-nome /mnt/seu-nome`
   - Verificar a montagem: `df -h`

8. **Subir um Apache no Servidor:**
   - Na instância EC2, executar `sudo yum install httpd -y`
   - Iniciar o Apache: `sudo systemctl start httpd`
   - Habilitar para início automático no boot: `sudo systemctl enable httpd`
   - Verificar status: `sudo systemctl status httpd`

9. **Criar um Script que Valide se o Serviço está Online e Envie o Resultado para o seu Diretório no NFS:**
   - Criar arquivo `sudo nano check_apache.sh`, inserir linhas:
     ```bash
     #!/bin/bash
     NFS_DIR=/mnt/seu-nome
     systemctl is-active --quiet httpd
     STATUS=$?
     if [ $STATUS -eq 0 ]; then
         echo "Apache está rodando" > $NFS_DIR/apache_status.txt
     else
         echo "Apache não está rodando" > $NFS_DIR/apache_status.txt
     fi
     ```
   - Salvar o arquivo, tornar o script executável: `sudo chmod +x check_apache.sh`
   - Executar o script: `sudo ./check_apache.sh`
   - Verificar o arquivo `.txt` com `cat /mnt/seu-nome/apache_status.txt`

10. **O Script deve Conter Data, Hora, Nome do Serviço, Status e Mensagem Personalizada de ONLINE ou OFFLINE:**
   - Alterar o script `check_apache.sh`, inserir linhas:
     ```bash
     #!/bin/bash
     NFS_DIR=/mnt/seu-nome
     SERVICE="Apache Server"
     systemctl is-active --quiet httpd
     STATUS=$?
     DATE_TIME=$(date '+%Y-%m-%d %H:%M:%S')
     if [ $STATUS -eq 0 ]; then
         STATUS_MSG="ONLINE"
         MESSAGE="O serviço HTTPD está rodando."
     else
         STATUS_MSG="OFFLINE"
         MESSAGE="O serviço HTTPD não está rodando."
     fi
     echo "$DATE_TIME - $SERVICE - $STATUS_MSG - $MESSAGE" > $NFS_DIR/apache_status.txt
     ```
   - Seguir os mesmos passos do tópico 8 após salvar o arquivo `check_apache.sh`

11. **O Script deve Gerar 2 Arquivos de Saída: 1 para o Serviço ONLINE e 1 para o Serviço OFFLINE:**
    - Alterar o script `check_apache.sh`, inserir linhas:
      ```bash
      #!/bin/bash
      NFS_DIR=/mnt/seu-nome
      SERVICE="Apache Server"
      systemctl is-active --quiet httpd
      STATUS=$?
      DATE_TIME=$(date '+%Y-%m-%d %H:%M:%S')
      if [ $STATUS -eq 0 ]; then
          STATUS_MSG="ONLINE"
          MESSAGE="O serviço HTTPD está rodando."
          OUTPUT_FILE="$NFS_DIR/apache_status_online.txt"
      else
          STATUS_MSG="OFFLINE"
          MESSAGE="O serviço HTTPD não está rodando."
          OUTPUT_FILE="$NFS_DIR/apache_status_offline.txt"
      fi
      echo "$DATE_TIME - $SERVICE - $STATUS_MSG - $MESSAGE" > $OUTPUT_FILE
      ```
    - Seguir os mesmos passos do tópico 8 após salvar o script
    - No diretório NFS devem constar três arquivos: `apache_status_offline.txt`, `apache_status_online.txt` e `apache_status.txt` (pode ser apagado)
    - Apagar arquivo: `rm /mnt/seu-nome/apache_status.txt`
    - Executar o script após parar ou reiniciar o serviço
    - Usar `cat /mnt/seu-nome/*` para exibir o conteúdo de saída de ambos arquivos

12. **Preparar a Execução Automatizada do Script a Cada 5 Minutos:**
    - Usar o Crontab: `sudo crontab -e`
    - Adicionar a linha: `*/5 * * * * /check_apache.sh`
    - Salvar e sair (pressionar "Esc" e digitar ":wq")
    - Listar tarefa crontab: `sudo crontab -l`
    - A cada cinco minutos as saídas do script serão alteradas de acordo com o estado do serviço

---
