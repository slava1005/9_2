## Домашнее задание к занятию «Вычислительные мощности. Балансировщики нагрузки»
### Задание 1. Yandex Cloud
### Что нужно сделать

1. Создать бакет Object Storage и разместить в нём файл с картинкой:
2. Создать бакет в Object Storage с произвольным именем (например, имя_студента_дата).
3. Положить в бакет файл с картинкой.
4. Сделать файл доступным из интернета.
5. Создать группу ВМ в public подсети фиксированного размера с шаблоном LAMP и веб-страницей, содержащей ссылку на картинку из бакета:
5. Создать Instance Group с тремя ВМ и шаблоном LAMP. Для LAMP рекомендуется использовать image_id = fd827b91d99psvq5fjit.
    Для создания стартовой веб-страницы рекомендуется использовать раздел user_data в meta_data.
6. Разместить в стартовой веб-странице шаблонной ВМ ссылку на картинку из бакета.
7. Настроить проверку состояния ВМ.
8. Подключить группу к сетевому балансировщику:
9. Создать сетевой балансировщик.
10. Проверить работоспособность, удалив одну или несколько ВМ.
11. (дополнительно)* Создать Application Load Balancer с использованием Instance group и проверкой состояния.

### Выполнение задания 1. Yandex Cloud
Создаем бакет в Object Storage с моими инициалами и текущей датой:
```
// Создаем сервисный аккаунт для backet
resource "yandex_iam_service_account" "service" {
  folder_id = var.folder_id
  name      = "bucket-sa"
}

// Назначение роли сервисному аккаунту
resource "yandex_resourcemanager_folder_iam_member" "bucket-editor" {
  folder_id = var.folder_id
  role      = "storage.editor"
  member    = "serviceAccount:${yandex_iam_service_account.service.id}"
  depends_on = [yandex_iam_service_account.service]
}

// Создание статического ключа доступа
resource "yandex_iam_service_account_static_access_key" "sa-static-key" {
  service_account_id = yandex_iam_service_account.service.id
  description        = "static access key for object storage"
}

// Создание бакета с использованием ключа
resource "yandex_storage_bucket" "savilovvv" {
  access_key = yandex_iam_service_account_static_access_key.sa-static-key.access_key
  secret_key = yandex_iam_service_account_static_access_key.sa-static-key.secret_key
  bucket = local.bucket_name
  acl    = "public-read"
}
```
За текущую дату в названии бакета будет отвечать локальная переменная current_timestamp в формате "день-месяц-год":
```
locals {
    current_timestamp = timestamp()
    formatted_date = formatdate("DD-MM-YYYY", local.current_timestamp)
    bucket_name = "savilovvv-${local.formatted_date}"
}
```
Код Terraform для создания бакета можно посмотреть в файле bucket.tf - https://github.com/slava1005/9_2/blob/main/terraform/bucket.tf

Загружу в бакет файл с картинкой:
```
resource "yandex_storage_object" "deadline-picture" {
  access_key = yandex_iam_service_account_static_access_key.sa-static-key.access_key
  secret_key = yandex_iam_service_account_static_access_key.sa-static-key.secret_key
  bucket = local.bucket_name
  key    = "flagimperii"
  source = "~/flagimperii.jpg"
  acl = "public-read"
  depends_on = [yandex_storage_bucket.savilovvv]
}
```
Источником картинки будет файл, лежащий в моей домашней директории, за публичность картинки будет отвечать параметр acl = "public-read".

Код Terraform для загрузки картинки можно посмотреть в файле upload_image.tf - https://github.com/slava1005/9_2/blob/main/terraform/upload_image.tf

Проверю созданный бакет:

![img1_1](https://github.com/user-attachments/assets/1a5b253f-68da-4c2c-bf88-7f7424ac4964)

![img1_2](https://github.com/user-attachments/assets/e82c566c-8015-4147-a139-c21939d737ac)

Бакет создан и имеет в себе один объект.

Создаю группу ВМ в public подсети фиксированного размера с шаблоном LAMP и веб-страницей, содержащей ссылку на картинку из бакета.
За сеть и подсеть public фиксированного размера будет отвечать код:
```
variable "default_cidr" {
  type        = list(string)
  default     = ["10.0.1.0/24"]
  description = "https://cloud.yandex.ru/docs/vpc/operations/subnet-create"
}

variable "vpc_name" {
  type        = string
  default     = "develop"
  description = "VPC network&subnet name"
}

variable "public_subnet" {
  type        = string
  default     = "public-subnet"
  description = "subnet name"
}

resource "yandex_vpc_network" "develop" {
  name = var.vpc_name
}

resource "yandex_vpc_subnet" "public" {
  name           = var.public_subnet
  zone           = var.default_zone
  network_id     = yandex_vpc_network.develop.id
  v4_cidr_blocks = var.default_cidr
}
```
За шаблон виртуальных машин с LAMP будет отвечать переменная в группе виртуальных машин image_id = "fd827b91d99psvq5fjit".

За создание стартовой веб-страницы будет отвечать параметр user_data в разделе metadata:
```
    user-data  = <<EOF
#!/bin/bash
cd /var/www/html
echo '<html><head><title>Picture</title></head> <body><h1>Look</h1><img src="http://${yandex_storage_bucket.savilovvv.bucket_domain_name}/deadline-cat.jpg"/></body></html>' > index.html
EOF
```
За проверку состояния виртуальной машины будет отвечать код:
```
  health_check {
    interval = 30
    timeout  = 10
    tcp_options {
      port = 80
    }
  }
```
Проверка здоровья будет выполняться каждые 30 секунд и будет считаться успешной, если подключение к порту 80 виртуальной машины происходит успешно в течении 10 секунд.

Полный код Terraform для создания группы виртуальных машин можно посмотреть в файле group_vm.tf - https://github.com/slava1005/9_2/blob/main/terraform/group_vm.tf

После применения кода Terraform получаем три настроенные по шаблону LAMP виртуальные машины:

![img1_3](https://github.com/user-attachments/assets/46930be3-e575-4f44-a456-0db414235b7a)

Создам сетевой балансировщик и подключу к нему группу виртуальных машин:
```
resource "yandex_lb_network_load_balancer" "balancer" {
  name = "lamp-balancer"
  deletion_protection = "false"
  listener {
    name = "http-check"
    port = 80
    external_address_spec {
      ip_version = "ipv4"
    }
  }
  attached_target_group {
    target_group_id = yandex_compute_instance_group.group-vms.load_balancer[0].target_group_id
    healthcheck {
      name = "http"
      interval = 2
      timeout = 1
      unhealthy_threshold = 2
      healthy_threshold = 5
      http_options {
        port = 80
        path = "/"
      }
    }
  }
}
```
Балансировщик нагрузки будет проверять доступность порта 80 и путь "/" при обращении к целевой группе виртуальных машин. 
Проверка будет выполняться с интервалом 2 секунды, с таймаутом 1 секунда. Пороговые значения для определения состояния 
сервера будут следующими: 2 неудачные проверки для перевода сервера LAMP в недоступное состояние и 5 успешных проверок для возврата в доступное состояние.

Проверю статус балансировщика нагрузки и подключенной к нему группе виртуальных машин после применения кода:

![img1_4](https://github.com/user-attachments/assets/10697b31-cd6f-424c-912d-cc0d12aa25db)

Балансировщик нагрузки создан и активен, подключенные к нему виртуальные машины в статусе "HEALTHY".

Проверю доступность сайта, через балансировщик нагрузки, открыв его внешний ip-адрес. Но для начала, нужно найти его внешний ip-адрес:

![img1_5](https://github.com/user-attachments/assets/67e079f1-4707-4bca-a2a2-26c0659380aa)

Открыв внешний ip-адрес балансировщика нагрузки я попадаю на созданную мной страницу:

![img1_6](https://github.com/user-attachments/assets/59820823-deb2-43d5-9693-bedad0b304d4)

Следовательно, балансировщик нагрузки работает.

Теперь нужно проверить, будет ли сохраняться доступность сайта после отключения пары виртуальных машин. Для этого выключаю две виртуальные машины из трех:

![img1_7](https://github.com/user-attachments/assets/3dd32cf5-e1f4-45ad-88c3-c9e03909f92f)

Сайт по прежнему доступен, так как одна из виртуальных машин продолжила работать и балансировщик нагрузки переключился на неё.

Через некоторое время, после срабатывания Healthcheck, выключенные виртуальные машины LAMP были заново запущены:

![img1_8](https://github.com/user-attachments/assets/c2270145-2dda-4cd9-9193-3ca093fdef46)

Таким образом доступность сайта была сохранена.

Полный код Terraform для создания сетевого балансировщика нагрузки можно посмотреть в файле network_load_balancer.tf - https://github.com/slava1005/9_2/blob/main/terraform/network_load_balancer.tf

Создаю Application Load Balancer с использованием Instance group и проверкой состояния.
Создаю целевую группу:
```
resource "yandex_alb_target_group" "application-balancer" {
  name           = "group-vms"

  target {
    subnet_id    = yandex_vpc_subnet.public.id
    ip_address   = yandex_compute_instance_group.group-vms.instances.0.network_interface.0.ip_address
  }

  target {
    subnet_id    = yandex_vpc_subnet.public.id
    ip_address   = yandex_compute_instance_group.group-vms.instances.1.network_interface.0.ip_address
  }

  target {
    subnet_id    = yandex_vpc_subnet.public.id
    ip_address   = yandex_compute_instance_group.group-vms.instances.2.network_interface.0.ip_address
  }
  depends_on = [
    yandex_compute_instance_group.group-vms
]
}
```
Создаю группу Бэкендов:
```
resource "yandex_alb_backend_group" "backend-group" {
  name                     = "backend-balancer"
  session_affinity {
    connection {
      source_ip = true
    }
  }

  http_backend {
    name                   = "http-backend"
    weight                 = 1
    port                   = 80
    target_group_ids       = [yandex_alb_target_group.alb-group.id]
    load_balancing_config {
      panic_threshold      = 90
    }    
    healthcheck {
      timeout              = "10s"
      interval             = "2s"
      healthy_threshold    = 10
      unhealthy_threshold  = 15 
      http_healthcheck {
        path               = "/"
      }
    }
  }
depends_on = [
    yandex_alb_target_group.alb-group
]
}
```
Создаю HTTP-роутер для HTTP-трафика и виртуальный хост:
```
resource "yandex_alb_http_router" "http-router" {
  name          = "http-router"
  labels        = {
    tf-label    = "tf-label-value"
    empty-label = ""
  }
}

resource "yandex_alb_virtual_host" "my-virtual-host" {
  name                    = "virtual-host"
  http_router_id          = yandex_alb_http_router.http-router.id
  route {
    name                  = "route-http"
    http_route {
      http_route_action {
        backend_group_id  = yandex_alb_backend_group.backend-group.id
        timeout           = "60s"
      }
    }
  }
depends_on = [
    yandex_alb_backend_group.backend-group
]
}
```
Создаю L7-балансировщик:
```
resource "yandex_alb_load_balancer" "application-balancer" {
  name        = "app-balancer"
  network_id  = yandex_vpc_network.develop.id

  allocation_policy {
    location {
      zone_id   = var.default_zone
      subnet_id = yandex_vpc_subnet.public.id
    }
  }

  listener {
    name = "listener"
    endpoint {
      address {
        external_ipv4_address {
        }
      }
      ports = [ 80 ]
    }
    http {
      handler {
        http_router_id = yandex_alb_http_router.http-router.id
      }
    }
  }

 depends_on = [
    yandex_alb_http_router.http-router
] 
}
```
За проверку состояния будет отвечать Healthcheck:
```
healthcheck {
      timeout              = "10s"
      interval             = "2s"
      healthy_threshold    = 10
      unhealthy_threshold  = 15 
      http_healthcheck {
        path               = "/"
      }
}
```
Проверю созданные ресурсы после применения кода:

![img1_9](https://github.com/user-attachments/assets/ce6b6425-d918-4e8a-a94d-e6dcc81340d5)

Все ресурсы создались.

Проверю, откроется ли сайт по внешнему адресу Application Load Balancer:

![img1_10](https://github.com/user-attachments/assets/3d6296c0-a509-4ef8-8393-3d6c7bedc52c)

Сайт открывается, Application Load Balancer работает.

После применения всего кода Terraform получаем следующий Output:

![img1_11](https://github.com/user-attachments/assets/9fbcb3b8-7ce9-4182-89a5-c2d5f52910f9)

Весь код Terraform можно посмотреть по ссылке: https://github.com/slava1005/9_2/tree/main/terraform
