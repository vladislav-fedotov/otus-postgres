1. сделать в GCE инстанс с Ubuntu 20.04
```gcloud compute instances create postgres-container \
        --zone=$ZONE \
        --image=$IMAGE \
        --image-project=ubuntu-os-cloud \
        --maintenance-policy=MIGRATE \
        --machine-type=$INSTANCE_TYPE \
        --boot-disk-size=10GB \
        --boot-disk-type=pd-ssd
```

2. поставить на нем Docker Engine
- Add Docker’s official GPG key:
`curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg`
- Use the following command to set up the stable repository:
```
echo \
  "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```
- Install Docker Engine:
```
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io
```
3. сделать каталог /var/lib/postgres
```
sudo mkdir -p /var/lib/postgres
```

4. развернуть контейнер с PostgreSQL 13 смонтировав в него /var/lib/postgres
```
sudo docker network create pg-net
sudo  docker run --name pg-server --network pg-net -e POSTGRES_PASSWORD=postgres -d -p 5432:5432 -v /var/lib/postgres:/var/lib/postgresql/data postgres:13
```

5. развернуть контейнер с клиентом postgres 
```
sudo docker run -it --rm --network pg-net --name pg-client postgres:13 psql -h pg-server -U postgres
```

6. подключится из контейнера с клиентом к контейнеру с сервером и сделать таблицу с парой строк
```
create database from_client_container;
\c from_client_container;
create table internal (msg text);
insert into internal values('long message');
```

7. подключится к контейнеру с сервером с ноутбука/компьютера извне инстансов GCP
```
gcloud compute instances stop postgres-container
gcloud compute instances add-tags postgres-container --tags=postgres
gcloud compute instances start postgres-container
```

Запустим контейнер `pg-server` и поменяем настройки доступа:
```
sudo docker start pg-server
sudo docker exec -it pg-server bash
apt-get update
apt-get install nano
su postgres
psql
ALTER USER postgres PASSWORD 'mySuperSTR0NGpa$$w0rd';
show hba_file;
exit
nano /var/lib/postgresql/data/pg_hba.conf
```

Подключемся с домашнего ноутбука к базе внутри контейнера:
```
psql -h 35.228.170.102 -U postgres -W
\l
```

8. удалить контейнер с сервером
```
sudo docker stop  pg-server
sudo docker rm pg-server
```

9. создать его заново 
```
sudo  docker run --name pg-server -e POSTGRES_PASSWORD=postgres -d -v /var/lib/postgres:/var/lib/postgresql/data postgres:13
sudo docker exec -it pg-server bash
su postgres
psql
\l
```

10. подключится снова из контейнера с клиентом к контейнеру с сервером
```
sudo docker stop  pg-server
sudo docker rm pg-server
sudo  docker run --name pg-server --network pg-net -e POSTGRES_PASSWORD=postgres -d -p 5432:5432 -v /var/lib/postgres:/var/lib/postgresql/data postgres:13
sudo docker run -it --rm --network pg-net --name pg-client postgres:13 psql -h pg-server -U postgres 
```

__Note__: пароль для подключения установленный ранее - mySuperSTR0NGpa$$w0rd!

11. проверить, что данные остались на месте
```
\c from_client_container;
\dt
select * from internal;
```

```
gcloud compute instances stop postgres-container
gcloud compute instances delete postgres-container
```