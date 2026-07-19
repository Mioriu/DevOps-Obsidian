## Variables

В конфиге из прошлых уроков все значения зашиты в ресурсы: тег образа, имя контейнера, порт. История знакомая по Ansible – чтобы один код работал для разных стендов, значения выносятся в переменные.

Переменная объявляется блоком `variable`. По соглашению все такие блоки складывают в отдельный файл `variables.tf` – как мы помним, все `.tf`-файлы директории читаются вместе:

HCL

Копировать

```hcl
# variables.tf
variable "nginx_version" {
  description = "Тег образа nginx"
  type        = string
  default     = "1.25"
}

variable "external_port" {
  description = "Порт на хосте"
  type        = number
  default     = 8080
}

variable "container_name" {
  type    = string
  default = "web"
}
```

`description` – пояснение для людей, `type` – тип значения (`string`, `number`, `bool`, есть и составные `list`, `map`), `default` – значение по умолчанию. Если `default` не задан, переменная обязательная: без значения `plan` остановится и запросит ввод.

В ресурсах переменная подставляется через `var.имя`:

HCL

Копировать

```hcl
# main.tf
resource "docker_image" "nginx" {
  name = "nginx:${var.nginx_version}"
}

resource "docker_container" "web" {
  name  = var.container_name
  image = docker_image.nginx.image_id

  ports {
    internal = 80
    external = var.external_port
  }
}
```

Обратите внимание на две формы записи. Когда переменная – все значение целиком, она пишется как есть, без кавычек: `name = var.container_name`. Когда подставляется внутрь строки – через `${...}`: `"nginx:${var.nginx_version}"`. Та же логика, что у ссылок на ресурсы из первого урока.

## Передача значений

Значение по умолчанию – самый слабый источник, его можно перекрыть несколькими способами.

![Приоритет источников значения переменной](https://prostodevops.ru/api/uploads/lessons/terraform-variables-outputs-modules-01.webp)Приоритет источников значения переменной

Основной – файл `terraform.tfvars` рядом с кодом, он подхватывается автоматически:

HCL

Копировать

```hcl
# terraform.tfvars
external_port  = 9090
container_name = "frontend"
```

Здесь только значения, без блоков `variable` – объявления остаются в `variables.tf`, а tfvars их заполняет. Обычно ситуация такая, что код один, а под каждый стенд свой файл значений, который подключается флагом – `terraform apply -var-file=prod.tfvars`.

Разовое значение передается флагом `-var`:

terraform apply -var="external_port=9191"

И четвертый способ – переменные окружения с префиксом `TF_VAR_`:

Bash

Копировать

```bash
export TF_VAR_external_port=9191
terraform apply
```

Этот способ – основной для секретов и CI: токен облака задается через `TF_VAR_token` в окружении и не появляется ни в одном файле. А чтобы секретное значение не светилось в выводе плана, у переменной выставляется `sensitive = true` – вместо значения будет печататься `(sensitive value)`.

По приоритету флаг `-var` сильнее tfvars, tfvars сильнее переменных окружения, и все они сильнее `default`.

## Outputs

После `apply` обычно нужны конкретные значения из созданного: IP новой машины, адрес балансировщика, имя контейнера.

![Outputs как интерфейс конфигурации наружу](https://prostodevops.ru/api/uploads/lessons/terraform-variables-outputs-modules-03.webp)Outputs как интерфейс конфигурации наружу

Копаться за ними в state или в консоли облака неудобно – для этого есть outputs:

HCL

Копировать

```hcl
# outputs.tf
output "container_name" {
  value = docker_container.web.name
}

output "external_port" {
  value = var.external_port
}
```

`value` – любое выражение: атрибут ресурса, переменная, их комбинация. После `apply` значения печатаются в конце сводки, а в любой момент достаются командой:

Bash

Копировать

```bash
terraform output                      # все outputs
terraform output external_port        # одно значение
terraform output -raw external_port   # без кавычек, для скриптов
```

Флаг `-raw` выводит голое значение без оформления – то, что подставляется в скрипты: `curl http://localhost:$(terraform output -raw external_port)`. По сути outputs – это интерфейс конфигурации наружу: их читают люди, скрипты и другие инструменты. Для секретных значений и тут есть `sensitive = true` – в сводке apply значение скроется, но `terraform output -raw` его отдаст.

## Data sources

Ресурс создает объект и владеет им.

![resource против data source: владение объектом](https://prostodevops.ru/api/uploads/lessons/terraform-variables-outputs-modules-02.webp)resource против data source: владение объектом

А data source – читает уже существующий, созданный кем-то другим: руками, другим Terraform-конфигом, вообще другой командой. Блок выглядит как ресурс, только со словом `data`:

HCL

Копировать

```hcl
data "docker_network" "existing" {
  name = "app_net"
}

resource "docker_container" "web" {
  name  = var.container_name
  image = docker_image.nginx.image_id

  networks_advanced {
    name = data.docker_network.existing.name
  }
}
```

Допустим, сеть `app_net` создана руками через `docker network create`. Data source находит ее по имени и отдает атрибуты, а ссылка на них пишется с префиксом `data.` – `data.тип.имя.атрибут`. Контейнер подключается к существующей сети, но сама сеть под управление не попадает: `terraform destroy` удалит контейнер, а сеть не тронет – конфиг ею не владеет, только читает.

В облаках data sources используются на каждом шагу. Классика – найти актуальный образ ОС, чтобы не зашивать его идентификатор в код:

HCL

Копировать

```hcl
data "yandex_compute_image" "ubuntu" {
  family = "ubuntu-2404-lts"
}

resource "yandex_compute_instance" "vm" {
  # ...
  boot_disk {
    initialize_params {
      image_id = data.yandex_compute_image.ubuntu.id
    }
  }
}
```

По семейству `ubuntu-2404-lts` находится свежий образ этой ветки, и его `id` уходит в диск машины. Тот же прием – для существующих сетей, подсетей, ключей: все, что создано вне вашего конфига, подтягивается через data, а не копируется в код руками.