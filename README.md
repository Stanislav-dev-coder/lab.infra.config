# lab.infra.config

Конфигурация учебного стенда из пяти виртуальных машин в приватной сети за роутером. На стенде работают GitLab с CI/CD, мониторинг и тестовое приложение.

## Цель и задачи

Цель: ручное создание и настройка инфраструктуры.

- Приватная сеть, выход в интернет через NAT-инстанс, доступ по SSH через бастион
- Единая точка входа по HTTPS с публичным wildcard-сертификатом
- Метрики и алерты со всех машин
- Пуш git-тега собирает образ и обновляет приложение
- Доступ между машинами ограничен матрицей правил

## Решения

- Роутер выполняет роли NAT-инстанса, SSH-бастиона и reverse proxy. TLS терминируется на nginx роутера, к бэкендам идёт HTTP
- Сертификат `*.lab.emptyvessel-playground.ru` выпускает Let's Encrypt через lego и DNS-01 в Timeweb Cloud, продление по таймеру systemd
- Сервисы и агенты работают в Docker Compose. На роутере Docker нет, сервисы установлены пакетами и бинарниками
- Монорепо с директорией на машину
- Мониторинг на Prometheus, Grafana и node_exporter. Дашборд, источник данных и правила Grafana Alerting загружаются из файлов provisioning
- CI на gitlab-runner с docker executor. Образы собираются через Docker-in-Docker и хранятся в Container Registry GitLab
- Деплой выполняет задача пайплайна, которая по SSH запускает `docker compose up -d --pull always` на app
- Сегментация на таблице nftables `ip segment` на каждой машине с Docker. Цепочка `prerouting` с приоритетом `-150` срабатывает раньше DNAT Docker

## Топология и доступ

| Сеть      | Подсеть      | Шлюз платформы | Назначение                             |
| --------- | ------------ | -------------- | -------------------------------------- |
| внешняя   | 10.10.0.0/24 | 10.10.0.1      | Выход роутера в интернет, вход с хоста |
| приватная | 10.20.0.0/24 | 10.20.0.1      | Все машины                             |

| Машина     | vCPU | RAM  | Swap | Диск  | Внешняя   | Приватная  | Роль                                     |
| ---------- | ---- | ---- | ---- | ----- | --------- | ---------- | ---------------------------------------- |
| router     | 1    | 1 ГБ | -    | 5 ГБ  | 10.10.0.2 | 10.20.0.2  | NAT, бастион, nginx, lego, node_exporter |
| app        | 1    | 1 ГБ | -    | 5 ГБ  | -         | 10.20.0.10 | Тестовое приложение                      |
| gitlab     | 6    | 6 ГБ | 2 ГБ | 20 ГБ | -         | 10.20.0.11 | GitLab CE, Container Registry            |
| monitoring | 2    | 2 ГБ | 1 ГБ | 15 ГБ | -         | 10.20.0.12 | Prometheus, Grafana                      |
| runner     | 2    | 2 ГБ | 2 ГБ | 15 ГБ | -         | 10.20.0.13 | gitlab-runner, docker executor           |

На всех машинах Ubuntu 26.04. У роутера `enp5s0` во внешней сети и `enp6s0` в приватной, у остальных машин `enp5s0` в приватной.

Требования к платформе

- Адреса машин закреплены резервированием DHCP
- DNS платформы `10.20.0.1` резолвит имена машин `<имя>.incus`
- Зона `*.lab.emptyvessel-playground.ru` резолвится только внутри стенда, во внешней сети в `10.10.0.2`, в приватной в `10.20.0.2`
- DHCP приватной сети объявляет шлюзом роутер `10.20.0.2`, NAT платформы для приватной сети выключен
- ACL приватной сети запрещает выход и вход в обход роутера
- На картах всех машин, кроме роутера, включена фильтрация MAC и IPv4 источника

Единая точка входа работает на nginx роутера, порты `80` и `443`. HTTP перенаправляется на HTTPS.

| Имя                                      | Бэкенд                  |
| ---------------------------------------- | ----------------------- |
| `gitlab.lab.emptyvessel-playground.ru`   | `gitlab.incus:80`       |
| `registry.lab.emptyvessel-playground.ru` | `gitlab.incus:5050`     |
| `grafana.lab.emptyvessel-playground.ru`  | `monitoring.incus:3000` |
| `app.lab.emptyvessel-playground.ru`      | `app.incus:8080`        |

SSH к машинам и git по SSH идут через роутер с `ProxyJump router`. sshd контейнера GitLab слушает порт `2222`.

Разрешённый доступ внутри приватной сети, остальное отбрасывается.

| Куда       | Порт           | Откуда     |
| ---------- | -------------- | ---------- |
| все машины | 22             | router     |
| все машины | 9100           | monitoring |
| monitoring | 3000           | router     |
| gitlab     | 80, 5050, 2222 | router     |
| app        | 8080           | router     |
| app        | 22             | runner     |

Роутер принимает SSH только из внешней сети, `80` и `443` из обеих сетей, `9100` от monitoring. В интернет он пропускает только приватную подсеть.

## Runbook

Runbook описывает сборку стенда с чистого хоста на Arch Linux. Команды хоста выполняются от пользователя, команды машин от root после `sudo -i`.

Порядок сборки

1. Хост: Incus, golden-образ, ключи, платформа
2. router
3. monitoring
4. gitlab
5. runner
6. Проект app в GitLab
7. app и первый деплой

### Incus

Каталог дисков пулов выносится в home до первого запуска демона.

```bash
sudo pacman -S incus
sudo pacman -S --asdeps cdrtools
ls /dev/kvm
mkdir -p ~/incus/disks
sudo mkdir -p /var/lib/incus
sudo ln -s $HOME/incus/disks /var/lib/incus/disks
sudo systemctl enable --now incus.socket incus.service
sudo usermod -aG incus-admin $USER
newgrp incus-admin
incus admin init --minimal
```

### Golden-образ

```bash
mkdir -p ~/Images/ubuntu-26.04
cd ~/Images/ubuntu-26.04
curl -fL -o rootfs.img https://cloud-images.ubuntu.com/releases/26.04/release/ubuntu-26.04-server-cloudimg-amd64.img
cat > metadata.yaml <<EOF
architecture: x86_64
creation_date: $(date +%s)
properties:
  description: Ubuntu 26.04 cloudimg golden
  os: ubuntu
  release: "26.04"
  variant: cloud
EOF
tar -cJf metadata.tar.xz metadata.yaml
incus image import metadata.tar.xz rootfs.img --alias ubuntu-26.04-golden
incus image list
```

### Ключи

| Ключ                 | Назначение                                     |
| -------------------- | ---------------------------------------------- |
| `~/.ssh/ubuntu-cloud`| Вход пользователя `ubuntu` на все машины       |
| `~/.ssh/lab-gitlab`  | git по SSH от имени root GitLab                |
| `~/.ssh/lab-deploy`  | Деплой из CI под пользователем `deploy` на app |

```bash
test -f ~/.ssh/ubuntu-cloud || ssh-keygen -t rsa -f ~/.ssh/ubuntu-cloud -N "" -C ubuntu-cloud
ssh-keygen -t ed25519 -f ~/.ssh/lab-gitlab -C lab-gitlab
ssh-keygen -t ed25519 -f ~/.ssh/lab-deploy -N "" -C gitlab-ci
```

Ключ `ubuntu-cloud` попадает на машины через cloud-init.

```bash
cat > ~/Images/ubuntu-26.04/user-data.yaml <<EOF
#cloud-config
users:
  - name: ubuntu
    sudo: ALL=(ALL) NOPASSWD:ALL
    shell: /bin/bash
    ssh_authorized_keys:
      - $(cat ~/.ssh/ubuntu-cloud.pub)
EOF
```

При пересборке машин старые ключи узлов удаляются из `known_hosts`.

```bash
ssh-keygen -R 10.10.0.2
for h in app.incus gitlab.incus monitoring.incus runner.incus '[gitlab.incus]:2222'; do ssh-keygen -R "$h"; done
```

### SSH на хосте

В `~/.ssh/config`

```text
Host router
    HostName 10.10.0.2
    User ubuntu
    IdentityFile ~/.ssh/ubuntu-cloud

Host app monitoring gitlab runner
    HostName %h.incus
    User ubuntu
    IdentityFile ~/.ssh/ubuntu-cloud
    ProxyJump router

Host gitlab.lab.emptyvessel-playground.ru
    HostName gitlab.incus
    Port 2222
    User git
    IdentityFile ~/.ssh/lab-gitlab
    ProxyJump router
```

Клон репозитория для правки конфигов

```bash
git clone git@github.com:Stanislav-dev-coder/lab.infra.config.git ~/Projects/pet/lab.infra.config
```

### Платформа

Пул, сети и ACL

```bash
incus storage create infra btrfs size=60GiB
incus network create lab_10 ipv4.address=10.10.0.1/24 ipv4.nat=true ipv6.address=none
incus network create lab_20 ipv4.address=10.20.0.1/24 ipv4.nat=false ipv4.dhcp.gateway=10.20.0.2 ipv6.address=none
incus network set lab_10 raw.dnsmasq='local=/lab.emptyvessel-playground.ru/
address=/lab.emptyvessel-playground.ru/10.10.0.2
bogus-priv'
incus network set lab_20 raw.dnsmasq='local=/lab.emptyvessel-playground.ru/
address=/lab.emptyvessel-playground.ru/10.20.0.2
bogus-priv'
incus network acl create lab20
incus network set lab_20 security.acls=lab20
```

Профили

```bash
incus profile create router
incus profile device add router eth0 nic network=lab_10 name=eth0
incus profile device add router eth1 nic network=lab_20 name=eth1
incus profile create internal
incus profile device add internal eth0 nic network=lab_20 name=eth0
```

Машины

```bash
USER_DATA="$(cat ~/Images/ubuntu-26.04/user-data.yaml)"
SWAP_1G="$(printf '#cloud-config\nswap:\n  filename: /swapfile\n  size: 1G\n')"
SWAP_2G="$(printf '#cloud-config\nswap:\n  filename: /swapfile\n  size: 2G\n')"

incus init ubuntu-26.04-golden router --vm -s infra -c limits.cpu=1 -c limits.memory=1GiB -d root,size=5GiB \
  -c cloud-init.user-data="$USER_DATA"
incus init ubuntu-26.04-golden app --vm -s infra -c limits.cpu=1 -c limits.memory=1GiB -d root,size=5GiB \
  -c cloud-init.user-data="$USER_DATA"
incus init ubuntu-26.04-golden gitlab --vm -s infra -c limits.cpu=6 -c limits.memory=6GiB -d root,size=20GiB \
  -c cloud-init.user-data="$USER_DATA" -c cloud-init.vendor-data="$SWAP_2G"
incus init ubuntu-26.04-golden monitoring --vm -s infra -c limits.cpu=2 -c limits.memory=2GiB -d root,size=15GiB \
  -c cloud-init.user-data="$USER_DATA" -c cloud-init.vendor-data="$SWAP_1G"
incus init ubuntu-26.04-golden runner --vm -s infra -c limits.cpu=2 -c limits.memory=2GiB -d root,size=15GiB \
  -c cloud-init.user-data="$USER_DATA" -c cloud-init.vendor-data="$SWAP_2G"

incus profile assign router router
for vm in app gitlab monitoring runner; do incus profile assign $vm internal; done

incus config device override router eth0 ipv4.address=10.10.0.2
incus config device override router eth1 ipv4.address=10.20.0.2
incus config device override app eth0 ipv4.address=10.20.0.10 security.ipv4_filtering=true
incus config device override gitlab eth0 ipv4.address=10.20.0.11 security.ipv4_filtering=true
incus config device override monitoring eth0 ipv4.address=10.20.0.12 security.ipv4_filtering=true
incus config device override runner eth0 ipv4.address=10.20.0.13 security.ipv4_filtering=true

incus start router monitoring gitlab runner app
incus list
ssh -o StrictHostKeyChecking=accept-new router cloud-init status --wait
```

Резолв зоны стенда на хосте через dnsmasq сети `lab_10`

```bash
sudo tee /etc/systemd/system/incus-dns-lab_10.service > /dev/null <<'EOF'
[Unit]
Description=Incus per-link DNS configuration for lab_10
BindsTo=sys-subsystem-net-devices-lab_10.device
After=sys-subsystem-net-devices-lab_10.device

[Service]
Type=oneshot
ExecStart=/usr/bin/resolvectl dns lab_10 10.10.0.1
ExecStart=/usr/bin/resolvectl domain lab_10 ~lab.emptyvessel-playground.ru
ExecStart=/usr/bin/resolvectl llmnr lab_10 no
ExecStart=/usr/bin/resolvectl mdns lab_10 no
ExecStopPost=/usr/bin/resolvectl revert lab_10
RemainAfterExit=yes

[Install]
WantedBy=sys-subsystem-net-devices-lab_10.device
EOF
sudo systemctl daemon-reload
sudo systemctl enable --now incus-dns-lab_10
resolvectl query --cache=no gitlab.lab.emptyvessel-playground.ru
```

### Клон репозитория на машине

Выполняется на каждой машине, в комментарии ключа указывается имя машины. Первое подключение к внутренним машинам возможно после настройки сети роутера.

```bash
ssh -o StrictHostKeyChecking=accept-new <машина>
sudo -i
```

Публичный ключ добавляется в GitHub в Settings → Deploy keys без права записи.

```bash
apt update && apt upgrade -y
apt install -y git
ssh-keygen -t ed25519 -f /root/.ssh/infra_deploy -N "" -C "<машина> deploy lab.infra.config"
cat /root/.ssh/infra_deploy.pub
cat >> /root/.ssh/config <<'EOF'
Host github.com
    User git
    IdentityFile /root/.ssh/infra_deploy
    IdentitiesOnly yes
EOF
chmod 600 /root/.ssh/config
ssh -T git@github.com
git clone git@github.com:Stanislav-dev-coder/lab.infra.config.git /opt/lab.infra.config
```

### Docker и сегментация

Выполняется на monitoring, gitlab, runner и app.

```bash
apt install -y ca-certificates curl
install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
chmod a+r /etc/apt/keyrings/docker.asc
tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Architectures: $(dpkg --print-architecture)
Signed-By: /etc/apt/keyrings/docker.asc
EOF
apt update
apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

```bash
M=/opt/lab.infra.config/<машина>
ln -s $M/nftables/nft-rules /usr/local/bin/nft-rules
systemctl link $M/nftables/nft-rules.service
systemctl enable --now nft-rules
nft list table ip segment
```

### router

Сеть, форвардинг и NAT

```bash
R=/opt/lab.infra.config/router
apt install -y nftables nginx
cp $R/network/60-lab20.yaml /etc/netplan/60-lab20.yaml
chmod 600 /etc/netplan/60-lab20.yaml
netplan apply
mkdir -p /etc/systemd/network/10-netplan-enp6s0.network.d
ln -sfn $R/network/dns.conf /etc/systemd/network/10-netplan-enp6s0.network.d/dns.conf
networkctl reload
ln -sfn $R/sysctl/60-forward.conf /etc/sysctl.d/60-forward.conf
sysctl --system
cp $R/nftables/nftables.conf /etc/nftables.conf
nft -c -f /etc/nftables.conf
systemctl enable --now nftables
```

Сертификат. Токен API создаётся в панели Timeweb Cloud и записывается в `timeweb.token`, адрес почты в `LEGO_EMAIL`.

```bash
cd /tmp
curl -fsSLO https://github.com/go-acme/lego/releases/download/v5.5.2/lego_v5.5.2_linux_amd64.tar.gz
curl -fsSLO https://github.com/go-acme/lego/releases/download/v5.5.2/lego_5.5.2_checksums.txt
sha256sum --ignore-missing -c lego_5.5.2_checksums.txt
tar -xzf lego_v5.5.2_linux_amd64.tar.gz lego
install -m 755 lego /usr/local/bin/lego
mkdir -m 700 /etc/lego
vi /etc/lego/timeweb.token
cp $R/lego/lego.env.example /etc/lego/lego.env
vi /etc/lego/lego.env
chmod 600 /etc/lego/timeweb.token /etc/lego/lego.env
systemctl link $R/lego/lego.service $R/lego/lego.timer
systemctl start lego
systemctl enable --now lego.timer
```

nginx

```bash
rm /etc/nginx/sites-enabled/default
ln -sfn $R/nginx/lab /etc/nginx/sites-enabled/lab
nginx -t && systemctl reload nginx
```

node_exporter

```bash
V=1.12.1
cd /tmp
curl -fsSLO https://github.com/prometheus/node_exporter/releases/download/v$V/node_exporter-$V.linux-amd64.tar.gz
curl -fsSLO https://github.com/prometheus/node_exporter/releases/download/v$V/sha256sums.txt
sha256sum --check --ignore-missing sha256sums.txt
tar -xzf node_exporter-$V.linux-amd64.tar.gz
install -m 0755 node_exporter-$V.linux-amd64/node_exporter /usr/local/bin/node_exporter
useradd --system --no-create-home --shell /usr/sbin/nologin node_exporter
systemctl link $R/node-exporter/node_exporter.service
systemctl enable --now node_exporter
```

### monitoring

В `.env` задаются пароль администратора и `GF_SERVER_ROOT_URL=https://grafana.lab.emptyvessel-playground.ru/`.

```bash
cd /opt/lab.infra.config/monitoring
cp .env.example .env
chmod 600 .env
vi .env
docker compose up -d
docker compose ps
```

### gitlab

```bash
cd /opt/lab.infra.config/gitlab
docker compose up -d
docker compose ps
docker exec infra-gitlab cat /etc/gitlab/initial_root_password
```

В интерфейсе GitLab выполняется вход под root со сменой пароля и отключается Auto DevOps в Admin → Settings → CI/CD. Публичный ключ `~/.ssh/lab-gitlab.pub` добавляется в User settings → SSH keys. Проверка с хоста

```bash
ssh -o StrictHostKeyChecking=accept-new -T git@gitlab.lab.emptyvessel-playground.ru
```

### runner

Runner создаётся в интерфейсе GitLab в Admin → CI/CD → Runners → New instance runner. Токен `glrt-` записывается в `CI_SERVER_TOKEN`.

```bash
cd /opt/lab.infra.config/runner
cp .env.example .env
chmod 600 .env
vi .env
docker compose run --rm gitlab-runner register \
  --non-interactive \
  --url https://gitlab.lab.emptyvessel-playground.ru \
  --name runner \
  --executor docker \
  --template-config /etc/gitlab-runner-template/template.toml
docker compose up -d
docker exec infra-gitlab-runner gitlab-runner verify
```

### Проект app

Пустой проект `app` без README создаётся в интерфейсе GitLab в New project → Create blank project. Код готовится на хосте.

```bash
mkdir ~/app && cd ~/app
git init -b main
git remote add origin ssh://git@gitlab.lab.emptyvessel-playground.ru:2222/root/app.git
vi main.go go.mod Dockerfile .gitlab-ci.yml
git add .
git commit -m "feat: Приложение со сборкой и деплоем по тегу"
git push -u origin main
```

`main.go`

```go
package main

import (
	"fmt"
	"net/http"
)

var version = "dev"

func main() {
	http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprintf(w, "Hello from app %s\n", version)
	})
	http.ListenAndServe(":8080", nil)
}
```

`go.mod`

```text
module app

go 1.27
```

`Dockerfile`

```dockerfile
FROM golang:1.27.2-alpine3.24 AS build
WORKDIR /src
COPY . .
ARG VERSION=dev
RUN CGO_ENABLED=0 go build -ldflags "-X main.version=${VERSION}" -o /app .

FROM alpine:3.24
COPY --from=build /app /app
ENTRYPOINT ["/app"]
```

`.gitlab-ci.yml`

```yaml
build:
  image: docker:29.9.0-cli
  services:
    - docker:29.9.0-dind
  variables:
    DOCKER_TLS_CERTDIR: "/certs"
  rules:
    - if: $CI_COMMIT_TAG
  script:
    - echo "$CI_JOB_TOKEN" | docker login -u "$CI_REGISTRY_USER" --password-stdin "$CI_REGISTRY"
    - docker build --provenance=false --push --build-arg VERSION=$CI_COMMIT_TAG -t $CI_REGISTRY_IMAGE:$CI_COMMIT_TAG -t $CI_REGISTRY_IMAGE:latest .

deploy:
  image: alpine:3.24
  needs: [build]
  rules:
    - if: $CI_COMMIT_TAG
  script:
    - apk add --no-cache openssh-client
    - chmod 600 "$SSH_PRIVATE_KEY"
    - ssh -i "$SSH_PRIVATE_KEY" -o UserKnownHostsFile="$SSH_KNOWN_HOSTS" deploy@app.incus "cd /opt/lab.infra.config/app && docker compose up -d --pull always app"
```

Ключ узла app снимается с роутера.

```bash
ssh router ssh-keyscan -t ed25519 app.incus
```

В проекте в Settings → CI/CD → Variables создаются переменные типа File. В `SSH_PRIVATE_KEY` записывается содержимое `~/.ssh/lab-deploy`, в `SSH_KNOWN_HOSTS` вывод `ssh-keyscan`. В Settings → Repository → Deploy tokens создаётся токен со scope `read_registry` для app.

### app

Логин в Registry выполняется deploy token. В `authorized_keys` записывается содержимое `~/.ssh/lab-deploy.pub` с хоста.

```bash
useradd --create-home --shell /bin/bash --groups docker deploy
su - deploy -c 'docker login registry.lab.emptyvessel-playground.ru'
install -d -m 700 -o deploy -g deploy /home/deploy/.ssh
vi /home/deploy/.ssh/authorized_keys
chown deploy:deploy /home/deploy/.ssh/authorized_keys
chmod 600 /home/deploy/.ssh/authorized_keys
cd /opt/lab.infra.config/app
docker compose up -d node_exporter
```

Первый деплой запускается тегом с хоста.

```bash
cd ~/app
git tag v1.0.0
git push origin v1.0.0
```

### Обновление конфигов

Изменения вносятся в репозиторий на хосте и отправляются в GitHub, затем применяются на машине.

```bash
cd /opt/lab.infra.config && git pull
cd <машина> && docker compose up -d
docker kill -s HUP infra-prometheus
nginx -t && systemctl reload nginx
systemctl restart nft-rules
```

`HUP` нужен только при изменении конфига Prometheus, перезагрузка nginx только на роутере.

### Проверка

```bash
curl -s -o /dev/null -w '%{http_code}\n' https://gitlab.lab.emptyvessel-playground.ru/users/sign_in
curl -s -o /dev/null -w '%{http_code}\n' https://grafana.lab.emptyvessel-playground.ru/api/health
curl -s -o /dev/null -w '%{http_code}\n' https://registry.lab.emptyvessel-playground.ru/v2/
curl -s https://app.lab.emptyvessel-playground.ru/
ssh -T git@gitlab.lab.emptyvessel-playground.ru
ssh monitoring "docker exec infra-prometheus wget -qO- 'localhost:9090/api/v1/query?query=up'"
```

Ожидаются коды `200`, `200`, `401`, версия приложения, приветствие GitLab и значение `1` у всех целей.
