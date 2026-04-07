Обычный rebase перемещает коммиты на новую базу, сохраняя их структуру. Но что если нужно не просто переместить историю, а переписать её? Множество мелких WIP-коммитов, неправильный порядок изменений, опечатки в сообщениях — всё это можно исправить с помощью интерактивного rebase. Это один из самых мощных инструментов Git для создания идеальной истории проекта перед публикацией.

### Что такое интерактивный rebase

**Интерактивный rebase** позволяет изменить серию коммитов: объединить несколько в один, изменить порядок, отредактировать сообщения или даже содержимое. В отличие от обычного rebase, который автоматически применяет все коммиты, интерактивный режим даёт полный контроль над каждым шагом процесса.

#### Как запускается интерактивный режим

```ruby
# Изменить последние 3 коммита
$ git rebase -i HEAD~3

# Изменить все коммиты после определённого
$ git rebase -i abc123

# Изменить все коммиты в feature-ветке относительно main
$ git rebase -i main
```

После запуска открывается текстовый редактор со списком коммитов и инструкциями. Это ваш "план действий", который Git будет выполнять.

### Анатомия интерактивного rebase

При запуске интерактивного rebase Git показывает список коммитов в обратном хронологическом порядке (старые сверху):

```cmake
pick f7f3f6d Change my name a bit
pick 310154e Update README formatting and add blame
pick a5f4a0d Add cat-file

# Rebase 710f0f8..a5f4a0d onto 710f0f8 (3 commands)
#
# Commands:
# p, pick <commit> = use commit
# r, reword <commit> = use commit, but edit the commit message
# e, edit <commit> = use commit, but stop for amending
# s, squash <commit> = use commit, but meld into previous commit
# f, fixup <commit> = like "squash", but discard this commit's log message
# x, exec <command> = run command (the rest of the line) using shell
# b, break = stop here (continue rebase later with 'git rebase --continue')
# d, drop <commit> = remove commit
# l, label <label> = label current HEAD with a name
# t, reset <label> = reset HEAD to a label
# m, merge [-C <commit> | -c <commit>] <label> [# <oneline>]
```

Каждая строка представляет коммит с командой, которую нужно выполнить. По умолчанию используется `pick` — просто применить коммит как есть.

#### Порядок выполнения

Git выполняет команды сверху вниз. Это важно понимать при изменении порядка коммитов или при squash — операции применяются последовательно, и результат одной влияет на следующую.

### Объединение коммитов (squash)

Одна из самых частых задач — объединение нескольких мелких коммитов в один осмысленный.

#### Базовый squash

Представим историю с множеством промежуточных коммитов:

```sql
$ git log --oneline -5
a5f4a0d Fix typo
310154e Add tests for authentication
f7f3f6d WIP: authentication
e3b8a1c Initial authentication implementation
9b5c3e2 Add user model
```

Чтобы объединить WIP и исправления в один коммит:

```ruby
$ git rebase -i HEAD~4

# Изменяем план:
pick e3b8a1c Initial authentication implementation
squash f7f3f6d WIP: authentication
squash 310154e Add tests for authentication
squash a5f4a0d Fix typo
```

После сохранения Git:

1. Применит первый коммит (e3b8a1c)
2. Объединит следующие три коммита с первым
3. Откроет редактор для создания нового сообщения коммита

#### Разница между squash и fixup

`squash` сохраняет сообщения всех коммитов и позволяет создать новое:

```delphi
# При squash откроется редактор с таким содержимым:
# This is a combination of 4 commits.
# This is the 1st commit message:

Initial authentication implementation

# This is the commit message #2:

WIP: authentication

# This is the commit message #3:

Add tests for authentication

# This is the commit message #4:

Fix typo
```

`fixup` автоматически отбрасывает сообщение коммита, оставляя только изменения:

```sql
pick e3b8a1c Initial authentication implementation
fixup f7f3f6d WIP: authentication
fixup 310154e Add tests for authentication
fixup a5f4a0d Fix typo
```

При fixup итоговый коммит будет иметь только сообщение первого коммита.

### Изменение порядка коммитов (reorder)

Иногда коммиты создаются в неправильном логическом порядке. Интерактивный rebase позволяет переупорядочить их.

#### Простое изменение порядка

```sql
# Исходный порядок
pick f7f3f6d Add database migration
pick 310154e Add model
pick a5f4a0d Add controller

# Логичнее было бы:
pick 310154e Add model
pick f7f3f6d Add database migration
pick a5f4a0d Add controller
```

Просто поменяйте строки местами в редакторе. Git применит коммиты в новом порядке.

#### Осторожность при изменении порядка

Изменение порядка может привести к конфликтам, если коммиты зависят друг от друга:

```applescript
# Опасное изменение порядка
pick a5f4a0d Использует функцию из file.js
pick 310154e Создаёт file.js  # Этот должен идти первым!
```

Git остановится с ошибкой при попытке применить коммит, использующий несуществующий файл. Придётся разрешать конфликты или отменять rebase.

### Редактирование коммитов (edit)

Команда `edit` позволяет остановиться на конкретном коммите и внести изменения — как в сообщение, так и в содержимое.

#### Изменение содержимого старого коммита

```sql
$ git rebase -i HEAD~3

# План действий:
pick f7f3f6d Add user service
edit 310154e Add user controller  # Останавливаемся здесь
pick a5f4a0d Add user routes
```

Git применит первый коммит, затем остановится:

```sql
Stopped at 310154e... Add user controller
You can amend the commit now, with

  git commit --amend

Once you are satisfied with your changes, run

  git rebase --continue
```

Теперь можно:

1. Внести изменения в файлы
2. Добавить их в индекс: `git add .`
3. Изменить коммит: `git commit --amend`
4. Продолжить rebase: `git rebase --continue`

#### Разделение коммита

Edit также используется для разделения большого коммита на несколько:

```shell
# Остановились на коммите для редактирования
$ git reset HEAD^  # Отменяем коммит, но сохраняем изменения

# Теперь создаём несколько коммитов
$ git add src/models/user.js
$ git commit -m "Add user model"

$ git add src/controllers/user.js
$ git commit -m "Add user controller"

$ git add tests/user.test.js
$ git commit -m "Add user tests"

$ git rebase --continue
```

### Изменение сообщений коммитов (reword)

Самая простая операция — изменение сообщения без изменения содержимого:

```sql
# План rebase
reword f7f3f6d Add autentication  # Исправим опечатку
pick 310154e Add authorization
pick a5f4a0d Add user permissions
```

Git остановится на помеченном коммите и откроет редактор с текущим сообщением. После сохранения продолжит выполнение плана.

### Удаление коммитов (drop)

Иногда коммит нужно полностью исключить из истории:

```nginx
# Удаляем коммит с отладочным кодом
pick f7f3f6d Add feature
drop 310154e Add debug logs  # Этот коммит исчезнет
pick a5f4a0d Clean up code
```

Можно также просто удалить строку с коммитом — эффект будет тот же.

### Продвинутые возможности

#### Выполнение команд (exec)

Можно выполнять произвольные команды между коммитами:

```sql
pick f7f3f6d Add authentication
exec npm test  # Проверяем, что тесты проходят
pick 310154e Add authorization
exec npm test  # Снова проверяем
pick a5f4a0d Add user permissions
```

Если команда завершится с ошибкой, rebase остановится.

#### Точки остановки (break)

```sql
pick f7f3f6d Add feature A
break  # Остановиться для ручной проверки
pick 310154e Add feature B
pick a5f4a0d Add feature C
```

Git остановится в указанной точке, позволяя выполнить любые действия перед продолжением.

### Типичные сценарии использования

#### Очистка истории перед merge

Перед отправкой pull request:

```sql
# История разработки
WIP: start feature
Add basic implementation
Fix bug
WIP: continue
Add tests
Fix typo
Refactor
Fix another bug
Code review fixes

# После интерактивного rebase
Implement user authentication with tests
Address code review feedback
```

#### Исправление коммита в середине истории

```ruby
# Нашли ошибку в коммите 5 шагов назад
$ git rebase -i HEAD~6

# Помечаем нужный коммит как edit
# Исправляем ошибку
# Продолжаем rebase
```

### Решение проблем при интерактивном rebase

#### Конфликты при изменении порядка

При перестановке коммитов часто возникают конфликты:

```shell
# Git останавливается с конфликтом
CONFLICT (content): Merge conflict in src/app.js
error: could not apply 310154e... Add feature

# Решаем конфликт
$ vim src/app.js
$ git add src/app.js
$ git rebase --continue
```

#### Потеря коммитов

Если случайно удалили нужный коммит:

```ruby
# Отменяем весь rebase
$ git rebase --abort

# Или используем reflog после завершения
$ git reflog
$ git cherry-pick <потерянный-коммит>
```

### Важные предостережения

Интерактивный rebase — мощный инструмент, но с ним легко создать проблемы:

- **Не изменяйте опубликованную историю** — это приведёт к проблемам у других разработчиков
- **Делайте backup важных веток** — `git branch backup/before-rebase`
- **Тестируйте после больших изменений** — переупорядочивание может сломать логику
- **Сохраняйте атомарность коммитов** — каждый коммит должен быть рабочим

### Автоматизация с помощью autosquash

Git предоставляет способ упростить squash через специальное именование коммитов:

```ruby
# Создаём основной коммит
$ git commit -m "Add authentication module"

# Создаём fixup коммит
$ git commit --fixup=HEAD  # или --fixup=abc123

# При rebase с --autosquash Git автоматически расставит fixup
$ git rebase -i --autosquash main
```

Git автоматически поместит fixup-коммиты после соответствующих коммитов с правильными командами.

## Итог

Интерактивный rebase превращает Git в машину времени для вашего кода. Он позволяет переписать историю: объединить черновые коммиты в осмысленные изменения, исправить старые ошибки без создания новых коммитов, упорядочить изменения в логической последовательности. Ключ к мастерству — понимание, что каждая операция создаёт новые коммиты, поэтому работать нужно только с неопубликованной историей. При правильном использовании интерактивный rebase помогает создавать профессиональную историю проекта, где каждый коммит рассказывает часть истории развития кода.