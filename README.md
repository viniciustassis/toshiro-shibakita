Docker: Utilização prática no cenário de Microsserviços
Denilson Bonatti, Instrutor - Digital Innovation One

Muito se tem falado de containers e consequentemente do Docker no ambiente de desenvolvimento. Mas qual a real função de um container no cenários de microsserviços? Qual a real função e quais exemplos práticos podem ser aplicados no dia a dia? Essas são algumas das questões que serão abordadas de forma prática pelo Expert Instructor Denilson Bonatti nesta Live Coding. IMPORTANTE: Agora nossas Live Codings acontecerão no canal oficial da dio._ no YouTube. Então, já corre lá e ative o lembrete! Pré-requisitos: Conhecimentos básicos em Linux, Docker e AWS.

No exemplo foi utilizada 3 máquinas virtuais EC2 criadas na AWS.

## 1. Em uma máquina foi iniciado o docker swarm.

Iniciar um cluster via docker swarm:
```sh
docker swarm init
```

Copiar o retorno do comando para que possa ser colado nos outros servidores.
```sh
docker swarm join --token [token_informado_apos_init_swarm] [ip:porta]
```

## 2. Depois acessar as outras 2 máquinas

Informar nos outros servidores que eles fazem parte do cluster.
```sh
docker swarm join --token [token_informado_apos_init_swarm] [ip:porta]
```

## 3. Checar os nós pertecentes ao cluster
No primeiro container executar:
```sh
docker node ls
```

## 4. Criar um serviço de containers
```sh
docker service create --name web-server --replicas 3 -dt -p 80:80 --mount type=volume,src=app,dst=/app/ webdevops/php-apache:alpine-php7
```

## 5. Verificar o serviço criado
```sh
docker service ps web-server
```

## 6. Criar réplica de volume no cluster
Os volumes não são replicados automaticamente

### 6.1 Criar um servidor NFS
```sh
apt-get install nfs-server
```

A primeira máquina virtual será o servidor NFS e os demais serão clientes do servidor NFS.

### 6.2 Criar cliente NFS
Nas demais máquinas virtuais
```sh
apt-get install nfs-common
```

### 6.3 Criar arquivo de configuração no servidor NFS
```sh
nano /etc/exports/

# Replicar uma pasta para os clientes
/var/lib/docker/volumes/app/_data *(rw,sync,subtree_check)
```

```sh
exportfs -ar
```

## 6.4 Verificar o que está compartilhado
```sh
showmount -e
```

## 6.4 Montar a pasta compartilhada nos clientes
```sh
mount -o v3 [ip:/var/lib/docker/volumes/app/_data] /var/lib/docker/volumes/app/_data
```

## 7. Criação de proxy
Quando uma requisição chegar na 1ª máquina deve ser replicada nos demais containers.
Pode-se usar o nginx

```sh
mkdir proxy
cd proxy
```

### Criar um arquivo de configuração e informar e quais IPs pertencem ao proxy.
```sh
nano ngingx.config
```

Com o seguinte conteúdo de exemplo:
```sh
http {
   
    upstream all {
        server 172.31.0.37:80;
        server 172.31.0.151:80;
        server 172.31.0.149:80;
    }

    server {
         listen 4500;
         location / {
              proxy_pass http://all/;
         }
    }

}
```

### Criar um arquivo dockerfile
Informar o arquivo de configuração criado para a imagem do nginx

### Criar uma imagem com a configuração criada do nginx
```sh
docker build -t proxy-app .
```

### Subir um container com a imagem criada
Apenas no 1º host, máquina virtual 1, rodar o container, Não precisa ser replicado.
```sh
docker container run --name my-proxy-app -dti -p 4500:4500 proxy-app
```