Развертку делал на Debian 11 от root

Обновили систему и поставили docker 
```bash
apt update && apt upgrade -y && apt install docker.io vim docker-compose -y
```

на самом сервере крайне желательно настроить /etc/hosts
а также на системе с которой подключаетесь 

в /etc/hosts сделать 


IP gitlab.example.com registry.example.com

на [windows ](https://www.howtogeek.com/784196/how-to-edit-the-hosts-file-on-windows-10-or-11/)



Проверяем занят ли 22 порт, если что-то вывело - делаем пункт с настройкой ssh
```bash
ss -tulpan | grep 22
```

### Настройка ssh

если что-то вывело - лучше ssh отредактировать. 

```bash
vim /etc/ssh/sshd_config
```

> [!Vim]
> Я использую vim, выйти их него если что 
> esc + :q! - без сохранения 
> esc + :wq - сохранить и выйти 
> Но можно использовать например nano

Находим строку Port и делаем как вот так 

```
Include /etc/ssh/sshd_config.d/*.conf  
  
Port 2222  
#AddressFamily any  
#ListenAddress 0.0.0.0  
#ListenAddress ::
```

рестарт sshd 
```bash
systemctl restart sshd
```

переподключились по ssh 
```bash
ssh USER@IP -p 2222
```

### Настройка gitlab
выполнять от пользователя root

Переходим в домашнюю директорию root и в ней создаем директорию gitlab

```bash
cd ~
mkdir gitlab
cd gitlab
```

Создаем файл `docker-compose.yml`
```bash
vim docker-compose.yml
```

```docker-compose.yml
version: '3.6'  
services:  
 gitlab:  
   image: gitlab/gitlab-ce:17.4.2-ce.0  
   container_name: gitlab  
   restart: always  
   hostname: 'gitlab.example.com'  
   environment:  
     GITLAB_OMNIBUS_CONFIG: |  
       # Add any other gitlab.rb configuration here, each on its own line  
       external_url 'https://gitlab.example.com'  
   ports:  
     - '80:80'  
     - '443:443'  
     - '22:22'  
   volumes:  
     - '/srv/gitlab/config:/etc/gitlab'  
     - '/srv/gitlab/logs:/var/log/gitlab'  
     - '/srv/gitlab/data:/var/opt/gitlab'  
   shm_size: '256m'  
   networks:  
     gitlab_net:  
       aliases:  
         - registry.example.com  
         - gitlab.example.com  
  
  
 gitlab-runner:  
   image: gitlab/gitlab-runner:alpine  
   container_name: gitlab-runner  
   restart: unless-stopped  
   depends_on:  
     - gitlab  
   volumes:  
     - /srv/gitlab-runner:/etc/gitlab-runner  
     - /var/run/docker.sock:/var/run/docker.sock  
   networks:  
     - gitlab_net  
  
networks:  
 gitlab_net:  
   driver: bridge
```


поднимаем gitlab
```bash
docker-compose up -d
```


Идем пить чай и ждем пока станет доступен сайт, а после загрузится форма авторизации. 

сайт поднялся через 6 минут,
админка через 7 минут после запуска

Получаем креды для root
```bash
docker exec -it gitlab grep 'Password:' /etc/gitlab/initial_root_password
```

авторизуемся и переходим на `/admin/users/root/edit`,
меняем пароль руту и проверяем его 

тут же можно создать нового юзера `/admin/users`

### Смена сертификатов

переходим в `/srv/gitlab/config/ssl`
Удаляем старые серты 
создаем новые 

```bash
cd /srv/gitlab/config/ssl
rm -rf *
openssl genrsa -out ca.key 2048

openssl req -new -x509 -days 3654 -key ca.key -subj "/C=CN/ST=GD/L=SZ/O=Acme, Inc./CN=Acme Root CA" -out ca.crt

openssl req -newkey rsa:2048 -nodes -keyout gitlab.example.com.key -subj "/C=CN/ST=GD/L=SZ/O=Acme, Inc./CN=*.example.com" -out gitlab.example.com.csr

openssl x509 -req -extfile <(printf "subjectAltName=DNS:example.com,DNS:www.example.com,DNS:registry.example.com,DNS:gitlab.example.com") -days 365 -in gitlab.example.com.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out gitlab.example.com.crt

```

копируем серт в раннер 
```bash
cp ca.crt /srv/gitlab-runner
```


### Активируем container registry

Нужно активировать его в файле `/srv/gitlab/config/gitlab.rb`, можно найти нужные строки в файле, но я предлагаю вариант немного проще

открываем файл `/srv/gitlab/config/gitlab.rb` на редактирование и вставляем с самый конец эти данные. ИМЕННО ДОПИСЫВАЕМ!!!
```bash
vim /srv/gitlab/config/gitlab.rb
```

```
registry_external_url 'https://registry.example.com'

gitlab_rails['registry_enabled'] = true
gitlab_rails['registry_host'] = "registry.example.com"

registry['enable'] = true
registry['registry_http_addr'] = "localhost:5000"
registry['log_directory'] = "/var/log/gitlab/registry"
registry['env_directory'] = "/opt/gitlab/etc/registry/env"
registry_nginx['enable'] = true
registry_nginx['listen_port'] = 443
registry_nginx['ssl_certificate'] = "/etc/gitlab/ssl/gitlab.example.com.crt"
registry_nginx['ssl_certificate_key'] = "/etc/gitlab/ssl/gitlab.example.com.key"
```



После чего перезапускаем gitlab 
```bash
docker restart gitlab
```

Ждем пока снова поднимется сайт (сейчас значительно быстрее будет)

авторизуемся и переходим на сайте `/admin/runners`
Создаем новый раннер и проставляем галочку `Run untagged jobs`
получили токен и регистрируем раннер 

```bash
docker exec -it gitlab-runner gitlab-runner register --url "https://gitlab.example.com" --tls-ca-file=/etc/gitlab-runner/ca.crt --registration-token "<token>"
```

Вместо `<token>` подставить свой токен 

> Enter an executor: docker, instance, custom, shell, ssh, parallels, docker-autoscaler, virtualbox, docker-windows, docker+machine, kubernetes:  
> `docker`  
> Enter the default Docker image (for example, ruby:2.7):  
> `alpine:latest`


---
Редактируем раннер 

```bash
vim /srv/gitlab-runner/config.toml
```

Редактируем 
`volumes = ["/cache"]` 
новое значение 
`volumes = ["/cache", "/var/run/docker.sock:/var/run/docker.sock"]`

Добавили `network_mode = "gitlab_gitlab_net"`

Мой конфиг выглядит примерно так

```
concurrent = 1  
check_interval = 0  
shutdown_timeout = 0  
  
[session_server]  
 session_timeout = 1800  
  
[[runners]]  
 name = "f56a876e1045"  
 url = "https://gitlab.example.com"  
 id = 3  
 token = "glrt-Envsnbkkesfd7qXaNydP"  
 token_obtained_at = 2024-11-11T10:34:55Z  
 token_expires_at = 0001-01-01T00:00:00Z  
 tls-ca-file = "/etc/gitlab-runner/ca.crt"  
 executor = "docker"  
 [runners.custom_build_dir]  
 [runners.cache]  
   MaxUploadedArchiveSize = 0  
   [runners.cache.s3]  
   [runners.cache.gcs]  
   [runners.cache.azure]  
 [runners.docker]  
   tls_verify = false  
   image = "alpine:latest"  
   privileged = false  
   disable_entrypoint_overwrite = false  
   oom_kill_disable = false  
   disable_cache = false  
   volumes = ["/cache", "/var/run/docker.sock:/var/run/docker.sock"]  
   shm_size = 0  
   network_mtu = 0  
   network_mode = "gitlab_gitlab_net"
```


### Настройка клиента для container registry
Создали проект и выбирает 

Deploy > Container Registry

там будут команды по типу 
```bash
ПРИМЕР
docker login gitlab.example.com
docker build -t gitlab.example.com/test/test .
docker push gitlab.example.com/test/test
```

заходим на систему где будем использовать docker 
обязательно заполняем /etc/hosts (в начале статьи есть пункт)

после чего 
```bash
vim /etc/docker/daemon.json
```
вставляем
```
{
  "insecure-registries" : ["registry.example.com"]
}
```

рестартим docker 
```bash
systemctl restart docker
```

авторизовали клиента 
```bash
docker login registry.example.com
```
логин и пароль от учетки гитлаба 


### Тестим полученный результат 

качаем контейнер с nginx
```bash
docker pull nginx:latest
```

Вернулись на страницу гитлаб где есть это 
docker login gitlab.example.com
docker build -t gitlab.example.com/test/test .
docker push gitlab.example.com/test/test

нас интересует третья строка 
вот эта часть gitlab.example.com/test/test

собираем команду 

```bash 
docker tag nginx:latest ЧАСТЬ_ВЫШЕ
```
вот так например
```bash 
docker tag nginx:latest registry.example.com/test/test:latest
```

после чего загружаем на gitlab 

```bash
docker push registry.example.com/test/test:latest
```
естественно если у вас другое имя - ставите свое 

----

заходим Build > Pipeline editor

пробуем запустить 
```.gitlab-cl.yml
stages:
  - scan

trivy_scan:
  stage: scan
  image:
    name: aquasec/trivy:latest
    entrypoint: [""]
  services: 
    - docker:dind
  script:
    - trivy image --scanners vuln --format json -o trivy-out-report.json --insecure "$CI_REGISTRY_IMAGE:latest"
  artifacts:
    paths:
      - trivy-out-report.json
```


Обращаю внимание, что он с первого раза может не запуститься. Советую запускать его несколько раз при возникновении ошибки. Как я понимаю он не может базу скачать. 











