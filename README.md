Docker: Utilização prática no cenário de Microsserviços
Live com o Intrutor: Denilson Bonatti.

# Sera usado o docker para implementar uma arquitetura de microserviços. 
## Para a replica dos volumes foi usado um servidor NFS
## Também utilizado um Proxy com Nginx e banco de dados Mysql


## Instalação dos programas
### container mysql 

```bash
docker pull mysql
```

```bash
docker run -e MYSQL_ROOT_PASSWORD=mysql123 --name mysql-A -d -p 3306:3306 mysql
```
Acessando o container

```bash
docker exec -it mysql-A bash
```
```bash
mysql -u root -p --protocol=tcp
```
Criando o banco
```bash
create database testedb;
use testedb;
```
Criando tabela
```bash
CREATE TABLE dados (
    AlunoID int,
    Nome varchar(50),
    Sobrenome varchar(50),
    Endereco varchar(150),
    Cidade varchar(50),
    Host varchar(50)
);
```

Criando diretório para o volume

<p>mkdir /var/lib/docker/volumes/app</p>
<p>mkdir /var/lib/docker/volumes/app/_data</p>
<p>nano /var/lib/docker/volumes/app/_data/index.php</p>

```bash
<html>

<head>
<title>Exemplo PHP</title>
</head>
<body>

<?php
ini_set("display_errors", 1);
header('Content-Type: text/html; charset=iso-8859-1');



echo 'Versao Atual do PHP: ' . phpversion() . '<br>';

$servername = "192.168.1.104";
$username = "root";
$password = "mysql123";
$database = "testedb";

// Criar conexão


$link = new mysqli($servername, $username, $password, $database);

/* check connection */
if (mysqli_connect_errno()) {
    printf("Connect failed: %s\n", mysqli_connect_error());
    exit();
}

$valor_rand1 =  rand(1, 999);
$valor_rand2 = strtoupper(substr(bin2hex(random_bytes(4)), 1));
$host_name = gethostname();


$query = "INSERT INTO dados (AlunoID, Nome, Sobrenome, Endereco, Cidade, Host) VALUES ('$valor_rand1' , '$valor_rand2', '$valor_rand2', '$valor_rand2', '$valor_rand2','$host_name')";


if ($link->query($query) === TRUE) {
  echo "New record created successfully";
} else {
  echo "Error: " . $link->error;
}

?>
</body>
</html>

```

# Testando Container Web-server
```bash
docker run --name web-server -dt -p 8080:80 --mount type=volume,src=app,dst=/app/ webdevops/php-apache:alpine-php7
```

```bash
docker rm --force web-server
```

# Criando Swarm
```bash
docker swarm init 
```
# Criando node nas outras maquinas com o Token - EX:
```bash
docker swarm join --token SWMTKN-sfdglk390kldpfgdf2s6qc3sq3q353uz27xbhçkkjdvlsfr95-5465dg456r523 192.168.0.5:2377
```
# Criar serviço 
```bash
docker service create --name web-server --replicas 5 -dt -p 80:80 --mount type=volume,src=app,dst=/app/ webdevops/php-apache:alpine-php7
```
# Verificando os serviços
```bash
docker service ps web-server
```
## Criando Servidor NFS 
instalar NFS no servidor 
```bash
apt-get install nfs-server
```
Instalar Agentes nas outras maquinas clientes
```bash
apt-get install nfs-common 
```
Editar arquivo de configuração do NFS (Como melhores praticas deve-se colocar apenas os IPs onde serão replicados... Neste exemplo o * representa todos)
```bash
nano /etc/exports
```
```bash
/var/lib/docker/volumes/app/_data *(rw,sync,subtree_check)
```
Após edição deve-se exportar 
```bash
exportfs -ar 
```
Verificando o diretório compartilhado
```bash
showmount -e
```
Adicionando nas maquinas clientes
```bash
mount -o v3 192.168.0.5:/var/lib/docker/volumes/app/_data /var/lib/docker/volumes/app/_data
```
## PROXY COM NGINX
Criar arquivo de configuração 
```bash
nano /proxy/nginx.conf
```

```bash
http {
   
    upstream all {
        server 192.168.0.5:80;
        server 192.168.1.90:80;
       
    }

    server {
         listen 4500;
         location / {
              proxy_pass http://all/;
         }
    }

}


events { }

```
# Criar um arquivo dockerfile
```bash
nano /proxy/dockerfile
```

```bash
FROM nginx
COPY nginx.conf /etc/nginx/nginx.conf
```
Gerar a imagem 
```bash
docker build -t proxy-app .
```
Criar o container com a imagem recem criada
```bash
docker container run --name my-proxy-app -dti -p 4500:4500 proxy-app
```

















