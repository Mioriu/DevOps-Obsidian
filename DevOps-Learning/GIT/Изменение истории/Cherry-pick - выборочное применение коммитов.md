Иногда нужно взять конкретное изменение из одной ветки и применить его в другой, не сливая ветки целиком. Исправление критического бага в production, перенос удачного решения между feature-ветками, применение hotfix к нескольким версиям продукта — все эти задачи решает cherry-pick. Это хирургический инструмент Git, позволяющий точечно переносить изменения между ветками.

### Что такое cherry-pick и как он работает

**Cherry-pick** — это операция применения изменений из конкретного коммита к текущей ветке. Название отражает суть: как при сборе вишни выбираются только спелые ягоды, так и cherry-pick позволяет выбрать только нужные коммиты из другой ветки.

#### Механика cherry-pick

Важно понимать: cherry-pick не перемещает коммит, а создаёт новый с теми же изменениями. Процесс выглядит так:

1. Git вычисляет diff (изменения) выбранного коммита относительно его родителя
2. Применяет эти изменения к текущей ветке
3. Создаёт новый коммит с тем же сообщением (но новым SHA-1)
4. Сохраняет ссылку на оригинальный коммит в метаданных

```applescript
# Исходное состояние
      D---E---F  (feature)
     /
A---B---C  (main)
         \
          G---H  (hotfix)

# Cherry-pick коммита H в main
$ git checkout main
$ git cherry-pick H

# Результат
      D---E---F  (feature)
     /
A---B---C---H'  (main)
         \
          G---H  (hotfix)
```

Коммит H' имеет те же изменения, что и H, но другой SHA-1 хеш, потому что у него другой родитель (C вместо G) и возможно другое время создания.

### Базовое использование cherry-pick

Рассмотрим основные команды и их поведение.

#### Применение одного коммита

```shell
# Найти нужный коммит
$ git log --oneline feature
e3b8c91 Fix critical security issue
7d3f2a8 Add experimental feature
4c9d0e1 Update documentation

# Переключиться на целевую ветку
$ git checkout main

# Применить конкретный коммит
$ git cherry-pick e3b8c91

# Git создаст новый коммит с теми же изменениями
[main 9f7e3d2] Fix critical security issue
 Date: Mon Oct 23 14:30:00 2023 +0300
 1 file changed, 5 insertions(+), 3 deletions(-)
```

По умолчанию cherry-pick сохраняет оригинальное сообщение коммита и авторство, но меняет коммиттера на текущего пользователя.

#### Опции cherry-pick

```ruby
# Применить изменения без создания коммита
$ git cherry-pick -n e3b8c91
# или
$ git cherry-pick --no-commit e3b8c91

# Изменения добавятся в рабочую директорию и индекс
# Можно внести дополнительные правки перед коммитом

# Добавить ссылку на оригинальный коммит
$ git cherry-pick -x e3b8c91
# В сообщение добавится строка:
# (cherry picked from commit e3b8c91...)
```

Опция `-x` полезна для отслеживания происхождения изменений, особенно при работе с несколькими ветками релизов.

### Cherry-pick нескольких коммитов

Часто нужно перенести не один коммит, а целую серию изменений.

#### Диапазон коммитов

```ruby
# Применить коммиты от A до B (не включая A)
$ git cherry-pick A..B

# Применить коммиты от A до B (включая A)
$ git cherry-pick A^..B

# Пример: перенести последние 3 коммита из feature
$ git cherry-pick feature~3..feature
```

Git применит коммиты по одному в хронологическом порядке. Если возникнет конфликт, процесс остановится.

#### Несколько отдельных коммитов

```ruby
# Перенести несколько конкретных коммитов
$ git cherry-pick abc123 def456 789ghi

# Коммиты применяются в указанном порядке
```

### Разрешение конфликтов при cherry-pick

Конфликты при cherry-pick возникают, когда изменения из переносимого коммита несовместимы с текущим состоянием файлов.

#### Процесс разрешения конфликтов

```sql
$ git cherry-pick e3b8c91
error: could not apply e3b8c91... Fix security issue
hint: after resolving the conflicts, mark the corrected paths
hint: with 'git add ' or 'git rm '
hint: and commit the result with 'git commit'
```

Когда возникает конфликт:

1. Git помечает конфликтные места в файлах стандартными маркерами
2. Процесс cherry-pick приостанавливается
3. Нужно вручную разрешить конфликты
4. Добавить исправленные файлы в индекс
5. Продолжить или отменить операцию

```shell
# После разрешения конфликтов
$ git add src/security.js

# Продолжить cherry-pick
$ git cherry-pick --continue

# Или отменить операцию
$ git cherry-pick --abort

# Пропустить проблемный коммит (при cherry-pick диапазона)
$ git cherry-pick --skip
```

#### Стратегии слияния

Можно указать стратегию для автоматического разрешения определённых конфликтов:

```ruby
# Предпочитать изменения из cherry-pick коммита
$ git cherry-pick -X theirs e3b8c91

# Предпочитать текущие изменения
$ git cherry-pick -X ours e3b8c91

# Использовать другую стратегию слияния
$ git cherry-pick --strategy=recursive -X patience e3b8c91
```

### Типичные сценарии использования

Cherry-pick решает специфические задачи, которые сложно решить другими способами.

#### Hotfix для нескольких версий

Критическое исправление нужно применить к нескольким версиям продукта:

```ruby
# Исправление сделано в main
$ git commit -m "Fix: SQL injection vulnerability"

# Применяем к версии 2.0
$ git checkout release/2.0
$ git cherry-pick main

# Применяем к версии 1.9
$ git checkout release/1.9
$ git cherry-pick main

# Каждая версия получает исправление независимо
```

#### Перенос изменений между feature-ветками

Коллега реализовал полезную утилиту в своей ветке:

```shell
# В ветке feature/payment есть полезный коммит
$ git log --oneline feature/payment
c3d4e5f Add currency conversion utility
b2c3d4e Update payment form
a1b2c3d Initial payment implementation

# Берём только утилиту в свою ветку
$ git checkout feature/shopping-cart
$ git cherry-pick c3d4e5f
```

#### Восстановление потерянных изменений

Случайно удалили нужные изменения при rebase:

```perl
# Находим потерянный коммит в reflog
$ git reflog
d4e5f6g HEAD@{5}: rebase: drop commit
c3d4e5f HEAD@{6}: commit: Important feature

# Восстанавливаем
$ git cherry-pick c3d4e5f
```

### Cherry-pick и история проекта

Важно понимать влияние cherry-pick на историю и возможные проблемы.

#### Дублирование коммитов

Главная опасность cherry-pick — создание дублирующих изменений:

```applescript
# Feature-ветка с cherry-picked коммитом
A---B---C---H'  (feature)

# Main с оригинальным коммитом
A---B---D---E---H  (main)

# При merge будут конфликты или дублирование
```

Git не понимает, что H и H' содержат одинаковые изменения, и попытается применить их дважды.

#### Отслеживание cherry-picked коммитов

Используйте опцию `-x` для документирования:

```csharp
$ git cherry-pick -x abc123

# В сообщении коммита появится:
Fix critical bug

(cherry picked from commit abc123def456...)
```

Это помогает понять происхождение изменений и избежать повторного применения.

### Продвинутые техники

#### Cherry-pick с изменениями

Иногда нужно адаптировать изменения под текущую ветку:

```ruby
# Применить без автоматического коммита
$ git cherry-pick -n abc123

# Внести необходимые изменения
$ vim src/adapted-file.js

# Закоммитить с новым сообщением
$ git commit -m "Adapt payment logic from feature branch

Based on commit abc123, adapted for current architecture"
```

#### Cherry-pick merge-коммитов

Merge-коммиты имеют несколько родителей, поэтому нужно указать, какую сторону использовать:

```python
# -m 1 использует первого родителя (обычно main)
$ git cherry-pick -m 1 merge-commit-hash

# -m 2 использует второго родителя (обычно feature-ветку)
```

#### Использование cherry-pick в скриптах

```bash
#!/bin/bash
# Скрипт для применения исправления ко всем release-веткам

FIX_COMMIT=$1
for branch in $(git branch -r | grep 'origin/release/'); do
    local_branch=${branch#origin/}
    git checkout $local_branch
    git pull origin $local_branch
    
    if git cherry-pick -x $FIX_COMMIT; then
        echo "Successfully applied to $local_branch"
        git push origin $local_branch
    else
        echo "Conflict in $local_branch, manual intervention needed"
        git cherry-pick --abort
    fi
done
```

### Альтернативы cherry-pick

В некоторых случаях есть лучшие альтернативы:

- **Merge** — когда нужны все изменения из ветки
- **Rebase** — для переноса целой серии коммитов с сохранением истории
- **Format-patch** — для переноса изменений между разными репозиториями
- **Общая ветка** — для изменений, нужных нескольким feature-веткам

### Диагностика проблем

#### Cherry-pick не применяется чисто

```shell
# Посмотреть, какие файлы изменяет коммит
$ git show --name-only abc123

# Проверить различия в контексте
$ git show abc123 -- path/to/file

# Попробовать с игнорированием пробелов
$ git cherry-pick -X ignore-space-change abc123
```

#### Поиск дублированных cherry-pick

```bash
# Найти коммиты с одинаковым patch-id
$ git log --all --pretty=format:"%H %s" | while read hash msg; do
    patch_id=$(git show $hash | git patch-id | awk '{print $1}')
    echo "$patch_id $hash $msg"
done | sort | uniq -d -w 40
```

## Итог

Cherry-pick — это точный инструмент для переноса конкретных изменений между ветками. Он создаёт новые коммиты с теми же изменениями, что позволяет применять исправления к нескольким версиям, делиться полезным кодом между feature-ветками и восстанавливать потерянные изменения. Ключевое понимание: cherry-pick не перемещает коммиты, а копирует изменения, что может привести к дублированию при последующих merge. Используйте cherry-pick для точечных операций, документируйте происхождение изменений опцией `-x` и помните о возможных конфликтах при слиянии веток с cherry-picked коммитами.