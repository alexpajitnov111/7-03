# «Подъём инфраструктуры в Yandex Cloud» - Алексей Пажитнов

### Задание 1 

**Выполните действия, приложите скриншот скриптов, скриншот выполненного проекта.**

От заказчика получено задание: при помощи Terraform и Ansible собрать виртуальную инфраструктуру и развернуть на ней веб-ресурс. 

В инфраструктуре нужна одна машина с ПО ОС Linux, двумя ядрами и двумя гигабайтами оперативной памяти. 

Требуется установить nginx, залить при помощи Ansible конфигурационные файлы nginx и веб-ресурса. 

Секретный токен от yandex cloud должен вводится в консоли при каждом запуске terraform.

Для выполнения этого задания нужно сгенирировать SSH-ключ командой ssh-keygen. Добавить в конфигурацию Terraform ключ в поле:

```
 metadata = {
    user-data = "${file("./meta.txt")}"
  }
``` 

В файле meta прописать: 
 
```
 users:
  - name: user
    groups: sudo
    shell: /bin/bash
    sudo: ['ALL=(ALL) NOPASSWD:ALL']
    ssh-authorized-keys:
      - ssh-rsa  xxx
```
Где xxx — это ключ из файла /home/"name_ user"/.ssh/id_rsa.pub. Примерная конфигурация Terraform:

```
terraform {
  required_providers {
    yandex = {
      source = "yandex-cloud/yandex"
    }
  }
}

variable "yandex_cloud_token" {
  type = string
  description = "Данная переменная потребует ввести секретный токен в консоли при запуске terraform plan/apply"
}

provider "yandex" {
  token     = var.yandex_cloud_token #секретные данные должны быть в сохранности!! Никогда не выкладывайте токен в публичный доступ.
  cloud_id  = "xxx"
  folder_id = "xxx"
  zone      = "ru-central1-a"
}

resource "yandex_compute_instance" "vm-1" {
  name = "terraform1"

  resources {
    cores  = 2
    memory = 2
  }

  boot_disk {
    initialize_params {
      image_id = "fd87kbts7j40q5b9rpjr"
    }
  }

  network_interface {
    subnet_id = yandex_vpc_subnet.subnet-1.id
    nat       = true
  }
  
  metadata = {
    user-data = "${file("./meta.txt")}"
  }

}
resource "yandex_vpc_network" "network-1" {
  name = "network1"
}

resource "yandex_vpc_subnet" "subnet-1" {
  name           = "subnet1"
  zone           = "ru-central1-b"
  network_id     = yandex_vpc_network.network-1.id
  v4_cidr_blocks = ["192.168.10.0/24"]
}

output "internal_ip_address_vm_1" {
  value = yandex_compute_instance.vm-1.network_interface.0.ip_address
}
output "external_ip_address_vm_1" {
  value = yandex_compute_instance.vm-1.network_interface.0.nat_ip_address
}
```

В конфигурации Ansible указать:

* внешний IP-адрес машины, полученный из output external_ ip_ address_ vm_1, в файле hosts;
* доступ в файле plabook *yml поля hosts.

```
- hosts: 138.68.85.196
  remote_user: user
  tasks:
    - service:
        name: nginx
        state: started
      become: yes
      become_method: sudo
```

Провести тестирование. 

---

**Решение**

1. main.tf
```
terraform {
  required_providers {
    yandex = {
      source = "yandex-cloud/yandex"
    }
  }
  required_version = ">= 0.13"
}

provider "yandex" {
  token = "y0_DkeNvQ9BulP4AC1TQq3htYAPjNXigfGIGDUDkeNvQ9BulP4AC1TQC1T"
  cloud_id = "b1ga3ko2a8tfk5f7tbh7"
  folder_id = "b1g8e5dbjk7p1t1o2hnr"
  zone = "ru-central1-a"
}

resource "yandex_compute_instance" "vm-2" {
  name                      = "linux-vm2"
  allow_stopping_for_update = true
  platform_id               = "standard-v2"


  resources {
    cores  = 2
    memory = 2
  }

  boot_disk {
    initialize_params {
      image_id = "fd8mchbe4poo11chk2gb"
    }
  }

  network_interface {
    subnet_id = "${yandex_vpc_subnet.subnet-2.id}"
    nat       = true
  }

  metadata = {
#    ssh-keys = "<имя_пользователя>:<содержимое_SSH-ключа>"
    user-data = file("./meta.yml")
  }
}

resource "yandex_vpc_network" "network-2" {
  name = "network2"
}

resource "yandex_vpc_subnet" "subnet-2" {
  name           = "subnet2"
  v4_cidr_blocks = ["192.168.10.0/24"]
  network_id     = "${yandex_vpc_network.network-2.id}"
}

```

2. meta.tf
```
#cloud-config
users:
- name: sadmin
  groups: sudo
  shell: /bin/bash
  sudo: ['ALL=(ALL) NOPASSWD:ALL']
  ssh-authorized-keys:
    - ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIH+9oszlN6WGIM0JYsocrINqldRceU/QTzZH4NSdjQRH root@test.ru-central1.internal
```
3. output.tf
```
output "ip-vm1-local" {
value = yandex_compute_instance.vm-1.network_interface.0.ip_address
}

output "ip-vm1-nat" {
value = yandex_compute_instance.vm-1.network_interface.0.nat_ip_address
}

```


4. ansible playbook: ng.yml
```
---
- hosts: test
  tasks:
    - name: Epel REPO
      become: true
      package:
        name: epel-release
        state: latest
    - name: Install nginx
      become: true
      package:
        name: nginx
        state: latest
    - name: Start nginx
      become: true
      service:
        name: nginx
        state: started
        enabled: yes
   
```
5. hosts:
```
[test]
sadmin@84.201.156.163

```

**Работоспособность:**

<img src = "https://github.com/alexpajitnov111/sdvps-homeworks/blob/main/img/07-1/01.png" width = 100%>

<img src = "https://github.com/alexpajitnov111/sdvps-homeworks/blob/main/img/07-1/02.png" width = 100%>

<img src = "https://github.com/alexpajitnov111/sdvps-homeworks/blob/main/img/07-1/03.png" width = 100%>

## Дополнительные задания* (со звёздочкой)

Их выполнение необязательное и не влияет на получение зачёта по домашнему заданию. Можете их решить, если хотите лучше разобраться в материале.лнить, если хотите глубже и/или шире разобраться в материале.

--- 
### Задание 2*

**Выполните действия, приложите скриншот скриптов, скриншот выполненного проекта.**

1. Перестроить инфраструктуру и добавить в неё вторую виртуальную машину. 
2. Установить на вторую виртуальную машину базу данных. 
3. Выполнить проверку состояния запущенных служб через Ansible.

--- 
### Задание 3*
Изучите [инструкцию](https://cloud.yandex.ru/docs/tutorials/infrastructure-management/terraform-quickstart) yandex для terraform.
Добейтесь работы паплайна с безопасной передачей токена от облака в terraform через переменные окружения. Для этого:

1. Настройте профиль для yc tools по инструкции.
2. Удалите из кода строчку "token = var.yandex_cloud_token". Terraform будет считывать значение ENV переменной YC_TOKEN.
3. Выполните команду export YC_TOKEN=$(yc iam create-token) и в том же shell запустите terraform.
4. Для того чтобы вам не нужно было каждый раз выполнять export - добавьте данную команду в самый конец файла ~/.bashrc

---

Дополнительные материалы: 

1. [Nginx. Руководство для начинающих](https://nginx.org/ru/docs/beginners_guide.html). 
2. [Руководство по Terraform](https://registry.terraform.io/providers/yandex-cloud/yandex/latest/doc). 
3. [Ansible User Guide](https://docs.ansible.com/ansible/latest/user_guide/index.html).
1. [Terraform Documentation](https://www.terraform.io/docs/index.html).

