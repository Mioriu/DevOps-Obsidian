## Зачем нужны модули

Прод и стейджинг обычно состоят из одного и того же набора ресурсов – различаются имена, размеры и порты. Без модулей этот набор копипастится: блоки образа и контейнера дублируются под каждое окружение, при добавлении третьего стенда копируются еще раз, и каждая правка дальше вносится во все копии. Стоит забыть одну – окружения разъехались, и стейджинг больше не повторяет прод. Та же болезнь, что была с плейбуками до ролей, и лекарство то же: модуль – упакованный набор ресурсов с переменными на входе и outputs на выходе, который вызывается сколько угодно раз с разными значениями.

## Что такое модуль

Технически модуль – это просто директория с `.tf`-файлами. Никакого специального формата: внутри обычные ресурсы, переменные и outputs. Директория, в которой запускается `terraform apply`, – тоже модуль, корневой. А переиспользуемые модули складывают в `modules/`:

Копировать

```
project/
├── main.tf
├── variables.tf
├── outputs.tf
└── modules/
    └── web_container/
        ├── main.tf
        ├── variables.tf
        └── outputs.tf
```

Внутри модуля, ресурсы в `main.tf`, переменные в `variables.tf`, outputs в `outputs.tf`. Переменные модуля – его вход, outputs – выход, а все, что между ними, снаружи не видно и не адресуется.

![Модуль: переменные на входе, outputs на выходе](https://prostodevops.ru/api/uploads/lessons/terraform-modules-pereispolzovanie-01.webp)Модуль: переменные на входе, outputs на выходе

Модуль подключается через интерфейс.

Соберем модуль из нашего постоянного примера – параметризованный nginx-контейнер:

HCL

Копировать

```hcl
# modules/web_container/variables.tf
variable "name" {
  type        = string
  description = "Имя контейнера"
}

variable "nginx_version" {
  type    = string
  default = "1.25"
}

variable "external_port" {
  type = number
}
```

HCL

Копировать

```hcl
# modules/web_container/main.tf
resource "docker_image" "nginx" {
  name = "nginx:${var.nginx_version}"
}

resource "docker_container" "this" {
  name  = var.name
  image = docker_image.nginx.image_id

  ports {
    internal = 80
    external = var.external_port
  }
}
```

HCL

Копировать

```hcl
# modules/web_container/outputs.tf
output "container_name" {
  value = docker_container.this.name
}
```

Все уже знакомое, просто лежит в отдельной директории. Имя ресурса `this` – распространенное соглашение для главного ресурса модуля: снаружи оно все равно не видно, а внутри читается как «тот самый контейнер этого модуля».

## Вызов модуля

Подключается модуль блоком `module` в корневом коде:

HCL

Копировать

```hcl
# main.tf
module "site_prod" {
  source        = "./modules/web_container"
  name          = "site-prod"
  external_port = 8080
}

module "site_staging" {
  source        = "./modules/web_container"
  name          = "site-staging"
  nginx_version = "1.27"
  external_port = 8081
}
```

`source` – откуда брать модуль, здесь локальный путь. Остальные аргументы – это переменные модуля: `name` и `external_port` без default обязательны, без них `plan` остановится с ошибкой. `nginx_version` у стейджинга переопределен, у прода взялся по умолчанию. После добавления модуля или смены `source` запускается `terraform init` – модули подтягиваются им же, как провайдеры.

Два вызова – два независимых набора ресурсов. В state они различаются префиксом модуля:

Bash

Копировать

```bash
$ terraform state list
module.site_prod.docker_container.this
module.site_prod.docker_image.nginx
module.site_staging.docker_container.this
module.site_staging.docker_image.nginx
```

Имена внутри модуля изолированы: оба контейнера называются `this`, и конфликта нет – полный адрес включает имя вызова. Правка модуля при этом затрагивает все вызовы сразу: поменяли внутренний порт в `modules/web_container/main.tf` – `plan` покажет изменения и по проду, и по стейджингу.

Провайдеры в дочерних модулях не настраиваются – блок `provider` живет в корне, модули используют его автоматически.

## Outputs модуля и связи

Значения изнутри модуля наружу достаются только через его outputs, по адресу `module.имя_вызова.имя_output`:

HCL

Копировать

```hcl
# outputs.tf в корне
output "prod_container" {
  value = module.site_prod.container_name
}
```

HCL

Копировать

```hcl
# modules/network/main.tf
variable "name" {
  type = string
}

resource "docker_network" "this" {
  name = var.name
}

output "name" {
  value = docker_network.this.name
}
```

В `web_container` добавляется переменная и подключение к сети:

HCL

Копировать

```hcl
# modules/web_container/variables.tf – дополнение
variable "network_name" {
  type = string
}
```

HCL

Копировать

```hcl
# modules/web_container/main.tf – дополнение в docker_container
  networks_advanced {
    name = var.network_name
  }
```

И в корне модули соединяются:

HCL

Копировать

```hcl
module "network" {
  source = "./modules/network"
  name   = "app-net"
}

module "site_prod" {
  source        = "./modules/web_container"
  name          = "site-prod"
  external_port = 8080
  network_name  = module.network.name
}
```

Ссылка `module.network.name` – это и зависимость: сеть создастся раньше контейнеров, как обычная зависимость по ссылке из первого урока.

## locals

Рядом с переменными есть locals – внутренние значения конфигурации:

HCL

Копировать

```hcl
locals {
  project = "myshop"
  sites = {
    prod    = 8080
    staging = 8081
  }
}
```

Отличие от variables: снаружи locals не передаются и не переопределяются, это константы и вычисляемые выражения для внутреннего пользования. Ссылка – через `local.имя`: `local.project`, `local.sites`. Удобны, когда одно значение или выражение нужно в нескольких местах кода, а выносить его в настраиваемую переменную незачем.

## for_each: окружения списком

Два блока `module` для двух окружений – нормально. Но окружений бывает пять, и блоки снова превращаются в копипаст, только уровнем выше. Для этого у блока `module` есть `for_each` – один блок разворачивается в несколько вызовов по элементам map:

![for_each: один блок разворачивается в несколько вызовов](https://prostodevops.ru/api/uploads/lessons/terraform-modules-pereispolzovanie-02.webp)for_each: один блок разворачивается в несколько вызовов

`for_each = local.sites` – по вызову на каждый ключ map. Внутри блока доступны `each.key` (имя ключа: `prod`, `staging`) и `each.value` (значение: порт). Добавить окружение теперь – одна строка в `local.sites`, и следующий `plan` покажет создание еще одного комплекта ресурсов.

Адресуются такие вызовы с ключом в скобках:

Bash

Копировать

```bash
$ terraform state list
module.site["prod"].docker_container.this
module.site["staging"].docker_container.this
...
```

И outputs читаются так же: `module.site["prod"].container_name`. Тот же `for_each`, кстати, работает и на обычных ресурсах – один блок `resource` на map значений, механика идентичная.

## Перенос существующих ресурсов в модуль

Самая частая ситуация с модулями в жизни – не новый проект, а рефакторинг: ресурсы уже созданы и работают, и их код выносится в модуль. И тут ждет сюрприз. Был ресурс `docker_container.web`, стал `module.site_prod.docker_container.this` – для учета это разные адреса. В state контейнер числится по старому, в коде объявлен по новому, и `plan` честно предлагает: старый удалить, новый создать. Для контейнера пережить можно, для прода с базой – нет.

Решение – блок `moved`, декларативное переименование в учете:

![Блок moved: перенос ресурса в модуль без пересоздания](https://prostodevops.ru/api/uploads/lessons/terraform-modules-pereispolzovanie-03.webp)Блок moved: перенос ресурса в модуль без пересоздания

С ним `plan` вместо удаления и создания показывает перенос: `docker_container.web has moved to module.site_prod.docker_container.this`, ноль изменений в реальной инфраструктуре. Блок едет в Git и применяется у всех, кто работает с этим кодом, а после применения везде его можно удалить. Ручная альтернатива – `terraform state mv` из урока про state, но она выполняется на одной машине и в истории кода не остается, поэтому для командной работы `moved` удобнее.

Правило отсюда простое: после любого рефакторинга, который двигает ресурсы между адресами, план читается особенно внимательно. Пара `-` и `+` на ресурсе, который никто не менял по сути, – признак смены адреса, а не реальных изменений.

## Источники модулей

Кроме локального пути, `source` принимает внешние источники. Публичный реестр – тот же [registry.terraform.io](http://registry.terraform.io/), где живут провайдеры:

HCL

Копировать

```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.8.1"
}
```

И Git-репозиторий, свой или чужой:

HCL

Копировать

```hcl
module "network" {
  source = "git::https://github.com/myorg/terraform-modules.git//network?ref=v1.2.0"
}
```

`//` отделяет путь внутри репозитория, `?ref=` закрепляет тег или коммит. Версию внешнего модуля фиксируют всегда – и по привычной причине воспроизводимости, и потому что чужой модуль – это код, который будет управлять вашей инфраструктурой: что в нем меняется от версии к версии, лучше контролировать.

В командах из этого складывается стандартная схема: типовые модули живут в отдельном репозитории `terraform-modules`, версии режутся тегами, проекты подключают их через `?ref=`. Обновление модуля – это осознанная смена тега в проекте с прогоном `plan`, а не внезапный сюрприз при очередном `init`.