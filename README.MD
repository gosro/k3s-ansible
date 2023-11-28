# Релизный репозиторий Ansible K3s, средства деплоя Kubernetes-кластеров

### <img src="images/warn.png" width="22" size="auto"> Примеры деплоя необходимых приложений находятся в директории [examples](https://git.vistgroup.ru/devops/platform/-/tree/master/examples)

   * [**Установка Ansible**](#%D1%83%D1%81%D1%82%D0%B0%D0%BD%D0%BE%D0%B2%D0%BA%D0%B0-ansible)
   * [**Доступные версии K3S**](#%D0%B4%D0%BE%D1%81%D1%82%D1%83%D0%BF%D0%BD%D1%8B%D0%B5-%D0%B2%D0%B5%D1%80%D1%81%D0%B8%D0%B8)
   * [**Деплой кластера**](#%D0%B4%D0%B5%D0%BF%D0%BB%D0%BE%D0%B9-%D0%BA%D0%BB%D0%B0%D1%81%D1%82%D0%B5%D1%80%D0%B0)
   * [**Как изменить ip адрес нод**](#%D0%BA%D0%B0%D0%BA-%D0%B8%D0%B7%D0%BC%D0%B5%D0%BD%D0%B8%D1%82%D1%8C-ip-%D0%B0%D0%B4%D1%80%D0%B5%D1%81-%D0%BD%D0%BE%D0%B4)
   * [**Установка kubectl**](#%D1%83%D1%81%D1%82%D0%B0%D0%BD%D0%BE%D0%B2%D0%BA%D0%B0-kubectl)
   * [**Установка helm**](#%D1%83%D1%81%D1%82%D0%B0%D0%BD%D0%BE%D0%B2%D0%BA%D0%B0-helm)
   * [**Установка ingress-nginx**](#%D1%83%D1%81%D1%82%D0%B0%D0%BD%D0%BE%D0%B2%D0%BA%D0%B0-ingress-nginx)
   * [**Установка metallb**](#%D1%83%D1%81%D1%82%D0%B0%D0%BD%D0%BE%D0%B2%D0%BA%D0%B0-metallb)

## Установка Ansible

> Ubuntu & Debian
```gradle  
sudo apt install ansible
```

> Fedora & Centos
```gradle  
sudo dnf install -y ansible    
```

> macOS
```gradle  
brew install ansible
```

## Доступные версии

>   *  v1.17.17+k3s1
>   *  v1.18.17+k3s1
>   *  v1.18.20+k3s1
>   *  v1.19.12+k3s1
>   *  v1.20.4+k3s1
>   *  v1.20.5+k3s1
>   *  v1.20.8+k3s1
>   *  v1.21.1+k3s1
>   *  v1.21.2+k3s1
>   *  v1.21.3+k3s1
>   *  v1.21.4+k3s1
>   *  v1.21.5+k3s2
>   *  v1.22.2+k3s2
>   *  v1.22.4+k3s1
>   *  v1.23.3+k3s1
>   *  v1.23.4+k3s1
>   *  v1.24.2+k3s1
>   *  v1.24.3+k3s1
>   *  **v1.24.4+k3s1**  << данную версию устанавливаем в продакшен.
>   *  v1.25.6+k3s1  
>   *  v1.26.0+k3s2
>   *  v1.26.1+k3s1
>   *  v1.26.3+k3s1

## Деплой кластера

Необходимые шаги:

1)  Описать имена машин и ip-адреса в hosts.ini файле `\inventory\hosts.ini`:

```ini
[cluster]
node-91 ansible_host=192.168.1.93
node-92 ansible_host=192.168.1.94
node-93 ansible_host=192.168.1.95

[master]
node-91
node-92
node-93

[worker]
node-91
node-92
node-93
```

При этом все хосты на которых ожидается запуск контейнеров должны быть перечислены в разделе worker, в том числе ноды с ролью мастер, если на них предполагается скедулинг подов.


2) Копируем конфигуарцию примера
```sh
cp -r inventory/example-cluster inventory/my-cluster
```
3) Меняем ansible_user в all.yaml в папке group_vars
```sh
sudo nano inventory/kgok-cluster/group_vars/all.yaml
```
```yaml
ansible_user: my-user

cluster_name: "oa-cluster.local"

k3s_version: v1.24.4+k3s1

# ipvs or iptables alowed
kubeproxy:
  - proxy-mode=iptables

kubeletargs: 
  - container-runtime=remote
  - container-log-max-files=2
  - container-log-max-size=10Mi

disable:
  - traefik
  - servicelb
```

Параметр "ansible_user" является пользователем по которму происходит ssh подключения по ключу на сервер, например если ваш юзер на сервере "my-user" то и "ansible_user" должен иметь идентичное значение

4) Запускаем деплой кластера

```sh
sudo ansible-playbook --key-file=/home/my-user/.ssh/id_rsa -i inventory/my-cluster/hosts.ini deploy.yaml -b
```
5) После успешного деплоя на хосте надо установить kubectl
```sh
curl -LO https://storage.googleapis.com/kubernetes-release/release/`curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt`/bin/linux/amd64/kubectl
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin/kubectl
```
6) Файл конфигураций лежит на целевом хосте в домашней дериктории:
```sh
cat /home/my-user/.kube/config
```

## Как изменить ip адрес нод

### Master node

- Меняем ip адреса на интерфейсах самих нод
- Останавливаем службу k3s
```gradle  
systemctl stop k3s
```

- Правим конфиг - /etc/rancher/k3s/cinfig.yaml 

```gradle  
    bind-adress: * ip адрес мастер ноды
    node-ip: * ip адрес данной ноды
```

- Генерируем новый сертификат

```gradle  

k3s certificate rotate
```
- Перечитываем конфиг etcd

```gradle  
k3s server --cluster-reset
```

- Стартуем службу k3s

```gradle  
systemctl start k3s
```

- Редактируем конфиг worker ноды

### Worker node

- Меняем ip адреса на интерфейсах самих нод
- Останавливаем службу k3s

```gradle  
systemctl stop k3s
```
- Правим конфиг -  /etc/rancher/k3s/cinfig.yaml 
- 
```gradle  
    server: * указываем ip адрес мастре ноды
    node-ip:  * ip адрес данной ноды
```

- Стартуем службу k3s

```gradle  
systemctl start k3s
```

### Установка kubectl

```gradle  
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
kubectl version --client
```

### Установка helm

```gradle  
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```

### Установка ingress-nginx

```gradle  
helm upgrade --install ingress-nginx ingress-nginx   --repo https://kubernetes.github.io/ingress-nginx   --namespace ingress-nginx --create-namespace
```

### Установка metallb

```gradle  
helm repo add metallb https://metallb.github.io/metallb
helm install metallb metallb/metallb --namespace metallb-system --create-namespace
```

### vi mlb.yaml

> - Меняет ip адресс в IPAddressPool
```
---
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: default-pool
  namespace: metallb-system
spec:
  addresses:
    - 10.14.255.120/32
  autoAssign: true
  avoidBuggyIPs: false

---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: l2-ip
  namespace: metallb-system
spec:
  ipAddressPools:
    - default-pool

```

### Deploy

```gradle  
kubectl -n metallb-system apply -f mlb.yaml

```