# Mongo em Docker

# Serviço
A arquitetura utilizada neste caso foi a de replica set, ou seja, há sempre um 
manager e um slave executando, a ideia inicial eramos ter três servidores, porém
no lab ter 2 servidores foi o suficiente, o ambiente ficou parecido com essa
imagem:

[![Fluxo](https://www.google.com/url?sa=i&source=images&cd=&cad=rja&uact=8&ved=2ahUKEwiD0rHZy9nhAhVkH7kGHXESCd8QjRx6BAgBEAU&url=https%3A%2F%2Fdocs.mongodb.com%2Fmanual%2Freplication&psig=AOvVaw2NA84eed-8OZ0qG1a1KDLO&ust=1555675152884691)](https://www.google.com/url?sa=i&source=images&cd=&cad=rja&uact=8&ved=2ahUKEwiD0rHZy9nhAhVkH7kGHXESCd8QjRx6BAgBEAU&url=https%3A%2F%2Fdocs.mongodb.com%2Fmanual%2Freplication&psig=AOvVaw2NA84eed-8OZ0qG1a1KDLO&ust=1555675152884691)

Em ambos os servidores temos Docker instalado e ambos os serviços do Mongo 
executam em containers. Abaixo o histórico de comandos utilizados para montar 
esse ambiente:


No server1
```
root@server1:/#mkdir /home/mongo/
root@server1:/#openssl rand -base64 741 >  /home/mongo/mongodb-keyfile
root@server1:/#chmod 600  /home/mongo/mongodb-keyfile 
root@server1:/#chown 999  /home/mongo/mongodb-keyfile
root@server1:/#docker run --name mongo -v /home/mongo/data:/data/db -v /home/mongo:/opt/keyfile --hostname="node1.mongodb.umbler.com" -p 40380:27017 -d mongo
root@server1:/#docker exec -it mongo /bin/bash
root@node1:/# mongo
> use admin
> db.createUser( {
     user: "ubadmin",
     pwd: "senha",
     roles: [ { role: "userAdminAnyDatabase", db: "admin" } ]
   });
> db.createUser( {
     user: "root",
     pwd: "senha",
     roles: [ { role: "root", db: "admin" } ]
   });
> exit
root@node1:/#exit
root@server1:/#docker rm -f mongo
```

Deve ser copiado o arquivo: /home/mongo/mongodb-keyfile para os demais servidores.

Em seguida é possível subir o primeiro container com mongo pronto para produção:

```docker run --name mongo -v /home/mongo/data:/data/db -v /home/mongo:/opt/keyfile --hostname="node01" --add-host node01:ip --add-host node02:ip -p 40380:40380 -d mongo --port 40380 --keyFile /opt/keyfile/mongodb-keyfile --replSet "rs0"```

Depois precisamos acessar este container para ativar a replica set:
```
root@server1:/#docker exec -it mongo /bin/bash
root@node1:/# mongo
> use admin
> db.auth("root", "SENHA_DE_ROOT");
> rs.initiate()
> rs0:PRIMARY> rs.conf()
```

No server2
Basta criar o segundo container ja pronto para produção:

```docker run --name mongo -v /home/mongo/data:/data/db -v /home/mongo:/opt/keyfile --hostname="node01" --add-host node01:ip --add-host node02:ip -p 40380:40380 -d mongo --port 40380 --keyFile /opt/keyfile/mongodb-keyfile --replSet "rs0"```

Depois disso precisamos voltar ao server1 e adicionar o segundo container ao replica set:

```
> rs.add("node2.mongodb.umbler.com")
> rs.status()
```

Dessa forma o replica set está configurado e pronto para ser utilizado, o "rs.status" retornará o status da replica.

# Backup
No server1 (que é o master) existem uma cron no seguinte formato:

```0 5 * * * /root/scritps/backup.sh```

Dentro do /root/scritps/ tem o backup.sh com o seguinte conteúdo:
```
#!/bin/bash
#Backup do banco emailcontext#
docker run --rm --name backup --link mongo:mongo -v /backup:/var/backup mongo mongodump --host mongo --port 40380 -u root -p'senha' --authenticationDatabase admin --db base --out /var/backup/`date +"%m-%d-%y"`
cd /backup
tar -cvzf server-$(date +"%m-%d-%y").tar.gz /backup/*
ftp -n -v ftpserver  <<EoF
        user user senha
        bin
        cd folder
        put server-$(date +"%m-%d-%y").tar.gz
EoF
rm /backup/* -Rf
```

Lembrando que a cada nova edição terá que ser adicionado outra linha de "docker run --rm" passando no parâmentro "--db" o nome do novo banco. 

**É possível melhorar esse processo 
utilizando um show dbs e usar um for para cada banco, mas isso fica para você, jovem padawã ;)


# Restore
Para restaurar o backup de algum banco, basta baixar do ftp, descompactar e rodar o seguinte comando:

```docker run --rm --name backup --link mongo:mongo -v /backup:/var/backup mongo mongorestore --host mongo --port 40380 -u root -p'senha'  /var/backup/mes-dia-ano/```
