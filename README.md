# Домашнее задание к занятию «Установка Kubernetes»

## ` Дмитрий Климов `

## Цель задания

###  * Установить кластер K8s.

## Чеклист готовности к домашнему заданию

###  * Развёрнутые ВМ с ОС Ubuntu 20.04-lts.

## Задание 1. Установить кластер k8s с 1 master node

1. Подготовка работы кластера из 5 нод: 1 мастер и 4 рабочие ноды.
2. В качестве CRI — containerd.
3. Запуск etcd производить на мастере.
4. Способ установки выбрать самостоятельно.


## Ответ:

## 1. Введение
Целью работы является создание отказоустойчивой инфраструктуры Kubernetes в облаке Yandex Cloud. Кластер состоит из 5 узлов (1 мастер и 4 воркера). Для автоматизации развертывания использован стек **Terraform + Kubespray (Ansible)**.

## 2. Инфраструктурный слой (Terraform)
Для обеспечения воспроизводимости инфраструктуры был написан манифест Terraform. Он создает VPC, подсеть и 5 инстансов на ОС Ubuntu 20.04 LTS.

### `main.tf:`

```hcl
terraform {
  required_providers {
    yandex = { source = "yandex-cloud/yandex" }
  }
}

provider "yandex" {
  token     = "OAUTH_TOKEN"
  cloud_id  = "CLOUD_ID"
  folder_id = "FOLDER_ID"
  zone      = "ru-central1-a"
}

resource "yandex_vpc_network" "k8s_network" { name = "k8s-network" }

resource "yandex_vpc_subnet" "k8s_subnet" {
  name           = "k8s-subnet"
  zone           = "ru-central1-a"
  network_id     = yandex_vpc_network.k8s_network.id
  v4_cidr_blocks = ["192.168.1.0/24"]
}

resource "yandex_compute_instance" "node" {
  for_each = { 
    "master": "192.168.1.10", 
    "worker1": "192.168.1.11", 
    "worker2": "192.168.1.12", 
    "worker3": "192.168.1.13", 
    "worker4": "192.168.1.14" 
  }
  name     = each.key
  resources { cores = 2; memory = 4 }
  boot_disk { initialize_params { image_id = "fd8vn6ra61c01hq58q75"; size = 20 } }
  network_interface { 
    subnet_id = yandex_vpc_subnet.k8s_subnet.id; 
    ip_address = each.value; 
    nat = true 
  }
  metadata = { ssh-keys = "ubuntu:${file("~/.ssh/id_rsa.pub")}" }
}
```
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/498b2ed1-7c27-4506-8f7a-452a4115a45e" />



## 3. Конфигурация кластера (`Kubespray Inventory`)

### Для развертывания `Kubernetes` выбран инструмент `Kubespray`. Конфигурация `inventory.ini` определяет роли узлов: `kube_control_plane` (управление), `etcd` (база данных) и `kube_node` (воркеры).

```ini
[all]
master  ansible_host=ВНЕШНИЙ_IP_MASTER ip=192.168.1.10
worker1 ansible_host=ВНЕШНИЙ_IP_W1 ip=192.168.1.11
worker2 ansible_host=ВНЕШНИЙ_IP_W2 ip=192.168.1.12
worker3 ansible_host=ВНЕШНИЙ_IP_W3 ip=192.168.1.13
worker4 ansible_host=ВНЕШНИЙ_IP_W4 ip=192.168.1.14

[all:vars]
ansible_user=ubuntu
ansible_ssh_common_args='-o StrictHostKeyChecking=no'

[kube_control_plane]
master

[etcd]
master

[kube_node]
worker1
worker2
worker3
worker4

[k8s_cluster:children]
kube_control_plane
kube_node
etcd
```
<img width="1920" height="1080" alt="Снимок экрана (2980)" src="https://github.com/user-attachments/assets/fa8862bf-ec11-4fed-812c-68322f545866" />


## 4. Технические характеристики реализации

  * Container Runtime: Используется containerd. Это стандарт индустрии, обеспечивающий высокую надежность.
  * Сетевой уровень (CNI): Установлен плагин Calico. Он обеспечивает маршрутизацию пакетов между подами на разных нодах         через BGP-протокол.
  * Хранилище etcd: Развернуто в режиме systemd-сервиса на мастер-ноде.

<img width="1920" height="1080" alt="Снимок экрана (2981)" src="https://github.com/user-attachments/assets/b4433856-b5fd-46ec-ae76-f0291b19ec62" />

<img width="1920" height="1080" alt="Снимок экрана (2983)" src="https://github.com/user-attachments/assets/e716335a-aab7-4dd9-b708-36c5bf057e88" />

<img width="1920" height="1080" alt="Снимок экрана (2984)" src="https://github.com/user-attachments/assets/3324e8f9-7f09-4697-92c6-3a7d55cb7498" />

## 5. Резюме

### Кластер готов к эксплуатации. Все требования задания выполнены:

  * Развернуто 5 ВМ.
  * Использован `containerd`.
  * ` etcd `локализован на мастер-ноде.
  * Кластер полностью управляется через ` kubectl `
















