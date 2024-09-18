
COMPASS - PB - SENAC/UNICESUMAR

Atividade de Linux / Prática

REQUISITOS AWS

1. CHAVE PÚBLICA PARA ACESSO AO AMBIENTE:

Serviço EC2 > Network & Security > Key Pairs > Create Key Pair > Tipo de chave para uso com Open SSH escolha ".pem" > Criar chave, O arquivo de chave privada é baixado automaticamente. Guardar em segurança para uso posterior no acesso a instância através de Open SSH;

2. Instância EC2:

Serviço EC2 > Instances > Launch instances > Name and tags - configurar de acordo com o projeto > Application and OS Images > "Amazon Linux 2 AMI (HVM) - Kernel 5.10, SSD Volume Type" > Tipo da instância > "t3.small" > Key pair > Selecionar chave criada anteriormente > Network settings - padrão da console (não modificar) > Configure Storage > Selecionar 16GB gp3 > Launch instance;

3. Elastic IP:

Serviço EC2 > EC2 Dashboard > Network & Security > Elastic Ips > Allocate Elastic IP address > Elastic IP address settings - padrão da console (não modificar) > Tags - configurar de acordo com o projeto > Allocate > Selecionar o endereço IP alocado > Actions > Associate Elastic IP address > Resource type - Instance > Escolher instância criada anteriormente > Associate;

4. Liberando portas de comunicação para acesso público:

Serviço EC2 > Instances > Network & Security > Security Groups > Selecionar SG do projeto > Inbound rules > Edit inbound rules > Add rule > Type - Custom TCP > Protocol - TCP > Port range - 111 > Source - Anywhere-IPv4 > Add rule > Type - Custom UDP > Protocol - UDP > Port range - 111 > Source - Anywhere-IPv4 > Add rule > Type - Custom TCP > Protocol - TCP > Port range - 2049 > Source - Anywhere-IPv4 > Add rule > Type - Custom UDP > Protocol - UDP > Port range - 2049 > Source - Anywhere-IPv4 > Add rule > Type - Custom TCP > Protocol - TCP > Port range - 80 > Source - Anywhere-IPv4 > Add rule > Type - Custom TCP > Protocol - TCP > Port range - 443 > Source - Anywhere-IPv4 > Save rules;

Requisitos no linux

5. Configurar o NFS (Network file system) entregue:

Usando uma VM com sistema Linux acessar a instância via OpenSSH > Copiar a chave ".pem" para a máquina virtual > Atribuir permissões de leitura a chave somente para o propietário do arquivo com o comando "chmod 400 path-to-your-key-pair.pem" > Conectando a instância com o comando "ssh -i caminho/para/sua-chave.pem ec2-user@seu-endereco-ip" substituindo o caminho/para/sua-chave.pem pelo caminho do arquivo de chave e ec2-user@seu-endereco-ip pelo nome de usuário e endereço IP da sua instância > Verificar distribuição Linux usada na instância pois os comandos podem ser diferentes - "cat /etc/os-release" > Em nosso caso a EC2 é compatível com CentOS, RHEL, and Fedora > Comando para o NFS "sudo yum install nfs-utils" -  retornou mensagem de que nessa instância o pacote já está instalado e na última versão > Verificar status do serviço - "sudo systemctl status nfs-server" > Iniciar o serviço NFS - "sudo systemctl start nfs-server";

6. Criar um diretorio dentro do filesystem do NFS com seu nome:

Use o comando "sudo mkdir /mnt/nfs_share" (diretorio para o ponto de montagem) substituindo nfs_share por seu nome > Configurar permissões no diretório compartilhado para todos - "sudo chmod 777 /mnt/seu-nome" > Editar arquivo de exportação (Definir permissões para clientes) - "sudo nano /etc/exports" > Adicionar linha no arquivo - "/mnt/seu-nome *(rw,sync,no_root_squash,insecure)" - salvar arquivo e aplicar "sudo exportfs -ra" para atualizar as exportações no NFS > Reiniciar o serviço NFS - "sudo systemctl restart nfs-server.service" >

No cliente, a VM com sistema Linux (Debian), execute o comando "sudo apt install nfs-common" para instalar o cliente NFS > Criar ponto de montagem - "sudo mkdir /mnt/seu-nome" > Montar diretorio compartilhado - "sudo mount -t nfs IP-da-instancia:/mnt/seu-nome /mnt/seu-nome" > Verificar a montagem - "df -h";

[TALVEZ - Para garantir que o diretório seja montado automaticamente após reinicializações, adicione a seguinte linha ao arquivo /etc/fstab no cliente:

192.168.1.100:/mnt/seu-nome /mnt/seu-nome nfs defaults 0 0]


7. Subir um apache no servidor - o apache deve estar online e rodando:

Na instância EC2 execute "sudo yum install httpd -y" > Iniciar o apache - "sudo systemctl start httpd" > Habilitar para inicio automático no boot - "sudo systemctl enable httpd" > Verificar status - "sudo systemctl status httpd";

8. Criar um script que valide se o serviço esta online e envie o resultado da validação para o seu diretorio no nfs:

Criar arquivo "sudo nano check_apache.sh", inserir linhas

#!/bin/bash

# Diretório NFS onde o resultado será salvo
NFS_DIR=/mnt/seu-nome

# Verifica o status do serviço Apache
systemctl is-active --quiet httpd
STATUS=$?

# Cria uma mensagem de status
if [ $STATUS -eq 0 ]; then
    echo "Apache está rodando" > $NFS_DIR/apache_status.txt
else
    echo "Apache não está rodando" > $NFS_DIR/apache_status.txt
fi
 
> Salvar o arquivo, tornar o script executável - "sudo chmod +x check_apache.sh" e executar o script "sudo ./check_apache.sh" > verificar o arquivo ".txt" com o comando "cat mnt/seu-nome/apache_status.txt" > Para testar o script retornando a mensagem "Apache está rodando" ou "Apache não está rodando" o mesmo deve ser executado após parar o serviço ou reiniciar o serviço;

9. O script deve conter - Data HORA + nome do serviço + Status + mensagem personalizada de ONLINE ou offline:

Alterar o scrip "check_apache.sh", apagar as linhas atuais e inserir as seguintes

!/bin/bash

# Diretório NFS onde o resultado será salvo
NFS_DIR=/mnt/seu-nome

# Nome do serviço
SERVICE="Apache Server"

# Verifica o status do serviço Apache
systemctl is-active --quiet httpd
STATUS=$?

# Data e hora atual
DATE_TIME=$(date '+%Y-%m-%d %H:%M:%S')

# Cria uma mensagem de status
if [ $STATUS -eq 0 ]; then
    STATUS_MSG="ONLINE"
    MESSAGE="O serviço HTTPD está rodando."
else
    STATUS_MSG="OFFLINE"
    MESSAGE="O serviço HTTPD não está rodando."
fi

# Escreve no arquivo apache_status.txt
echo "$DATE_TIME - $SERVICE - $STATUS_MSG - $MESSAGE" > $NFS_DIR/apache_status.txt

> Seguir os mesmos passos do tópico 8 após salvar o arquivo "check_apache.sh";

10. O script deve gerar 2 arquivos de saida: 1 para o serviço online e 1 para o serviço OFFLINE:

Alterar o scrip "check_apache.sh", apagar as linhas atuais e inserir as seguintes

#!/bin/bash

# Diretório NFS onde o resultado será salvo
NFS_DIR=/mnt/seu-nome

# Nome do serviço
SERVICE="Apache Server"

# Verifica o status do serviço Apache
systemctl is-active --quiet httpd
STATUS=$?

# Data e hora atual
DATE_TIME=$(date '+%Y-%m-%d %H:%M:%S')

# Cria uma mensagem de status
if [ $STATUS -eq 0 ]; then
    STATUS_MSG="ONLINE"
    MESSAGE="O serviço HTTPD está rodando."
    OUTPUT_FILE="$NFS_DIR/apache_status_online.txt"
else
    STATUS_MSG="OFFLINE"
    MESSAGE="O serviço HTTPD não está rodando."
    OUTPUT_FILE="$NFS_DIR/apache_status_offline.txt"
fi

# Escreve no arquivo de saída correspondente
echo "$DATE_TIME - $SERVICE - $STATUS_MSG - $MESSAGE" > $OUTPUT_FILE

> Mesmos passos do tópico 8 após salvar o script > No diretório NFS deve constar três arquivos - "apache_status_offline.txt", "apache_status_online.txt" e "apache_status.txt" (corresponde ao script anterior, pode ser apagado) > Apagar arquivo - "rm /mnt/seu-nome/apache_status.txt" > Executar o script após parar ou reiniciar o serviço > Usar "cat mnt/seu-nome/*" (* vai listar todos) para exibir o conteúdo de saída de ambos arquivos;

11. Preparar a execução automatizada do script a cada 5 minutos:

Para programar tarefas automatizadas usar o Crontab (cron table) - "sudo crontab -e" > Adicionar a linha no editor Vim - "*/5 * * * * /check_apache.sh" - salvar e sair - pressionar "Esc" e digitar ":wq" > Listar tarefa crontab - "sudo crontab -l" - a linha "*/5 * * * * /check_apache.sh" é exibida > A cada cinco minutos as saidas do script serão alteradas de acordo com o estado do serviço;

