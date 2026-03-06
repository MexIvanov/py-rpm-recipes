# Пример 4: Проект "местное время" tz-cli, сборка через Setuptools

### Python-программа (Модульная структура, сборка через Setuptools)

Использование системных утилит Linux (`timedatectl` и `date`) через Python (`subprocess`) вместо внешних API — это классический подход для системного администрирования.

Ниже переписаны оба примера. Теперь скрипты проверяют существование зоны через `timedatectl list-timezones` и получают локальное время этой зоны через команду `date` с подменой переменной окружения `TZ`.

---

В этом примере мы создадим утилиту `tz-cli`, которая состоит из нескольких файлов. Так как мы используем только встроенные модули Python (`subprocess`, `sys`, `argparse`), нам не понадобятся сторонние зависимости в RPM.

---

В этом примере мы используем только встроенные библиотеки Python и системные утилиты Linux, поэтому файл `requirements.txt` нам не нужен. Программа разделена на логическое ядро и интерфейс командной строки.

**Структура исходного кода перед архивацией (`tar -czvf`):**

```text
~/tz-cli-1.0/                       # Корневая папка проекта (версия 1.0)
│
├── setup.py                        # Инструкция для setuptools (как собирать и делать команду tz-cli)
│
└── tz_cli/                         # Главная папка Python-пакета (название с нижним подчеркиванием)
    │
    ├── __init__.py                 # Пустой файл, говорит Питону: "эта папка — модуль"
    │
    ├── core.py                     # Файл с логикой: функции для вызова timedatectl и date
    │
    └── cli.py                      # Файл интерфейса: парсинг аргументов (argparse) и print()
```

---


#### Шаг 1. Создание структуры проекта

```bash
mkdir -p ~/tz-cli-1.0/tz_cli
cd ~/tz-cli-1.0
```

**1. Файл `tz_cli/__init__.py`** (пустой файл)
```bash
touch tz_cli/__init__.py
```

**2. Файл `tz_cli/core.py` (Взаимодействие с Linux)**
Этот модуль отвечает за вызов системных команд CentOS.
```python
# tz_cli/core.py
import subprocess
import os

def get_valid_timezones():
    # Вызываем системную команду CentOS
    result = subprocess.run(['timedatectl', 'list-timezones'], capture_output=True, text=True)
    if result.returncode != 0:
        return []
    # Возвращаем список зон (каждая с новой строки)
    return result.stdout.splitlines()

def get_time_for_tz(tz_name):
    # Копируем текущее окружение и подменяем TimeZone
    env = os.environ.copy()
    env['TZ'] = tz_name
    
    # Вызываем Linux команду: date "+%Y-%m-%d %H:%M:%S"
    result = subprocess.run(
        ['date', '+%Y-%m-%d %H:%M:%S'], 
        env=env, 
        capture_output=True, 
        text=True
    )
    return result.stdout.strip()
```

**3. Файл `tz_cli/cli.py` (Интерфейс пользователя)**
```python
# tz_cli/cli.py
import argparse
from .core import get_valid_timezones, get_time_for_tz

def main():
    parser = argparse.ArgumentParser(description="Узнать время в указанной временной зоне.")
    parser.add_argument("timezone", nargs="?", default="UTC", help="Например: Europe/London или Asia/Tokyo")
    args = parser.parse_args()

    tz_target = args.timezone
    valid_zones = get_valid_timezones()

    if valid_zones and tz_target not in valid_zones:
        print(f"Ошибка: Временная зона '{tz_target}' не найдена в системе.")
        print("Используйте 'timedatectl list-timezones' для просмотра доступных зон.")
        return

    print(f"Запрашиваем время для зоны: {tz_target}...")
    current_time = get_time_for_tz(tz_target)
    print(f"[{tz_target}] Текущее время: {current_time}")

if __name__ == "__main__":
    main()
```

**4. Файл сборки `setup.py`**
```python
# setup.py
from setuptools import setup, find_packages

setup(
    name="tz-cli",
    version="1.0",
    packages=find_packages(),
    entry_points={
        "console_scripts": [
            "tz-cli=tz_cli.cli:main", 
        ]
    }
)
```

#### Шаг 2. Создание архива и SPEC-файла

```bash
cd ~
tar -czvf tz-cli-1.0.tar.gz tz-cli-1.0/
mv tz-cli-1.0.tar.gz ~/rpmbuild/SOURCES/
nano ~/rpmbuild/SPECS/tz-cli.spec
```

**Содержимое `tz-cli.spec`:**
```specfile
Name:           tz-cli
Version:        1.0
Release:        1%{?dist}
Summary:        CLI tool to get time by timezone using Linux commands
License:        MIT
Source0:        %{name}-%{version}.tar.gz
BuildArch:      noarch

BuildRequires:  python3-devel
BuildRequires:  python3-setuptools
Requires:       python3
# systemd содержит утилиту timedatectl
Requires:       systemd
# coreutils содержит утилиту date
Requires:       coreutils

%description
A modular Python tool that uses timedatectl and date 
to show time for a specific timezone.

%prep
%setup -q

%build
%py3_build

%install
%py3_install

%files
%{_bindir}/tz-cli
%{python3_sitelib}/tz_cli/
%{python3_sitelib}/tz_cli-*.egg-info/

%changelog
* Fri Mar 06 2026 Admin - 1.0-1
- Initial release
```

**Сборка и проверка:**
```bash
rpmbuild -ba ~/rpmbuild/SPECS/tz-cli.spec
sudo dnf localinstall ~/rpmbuild/RPMS/noarch/tz-cli-1.0-1.*.rpm -y

tz-cli Europe/London
```

---

### Структура директории в рабочем пространстве CentOS (`~/rpmbuild/`) для проекта tz-cli

После того как вы упаковали исходники в `.tar.gz` и написали `.spec` файлы, ваша стандартизированная сборочная директория `rpmbuild` должна выглядеть именно так:

```text
~/rpmbuild/
│
├── BUILD/                          # (Папка используется автоматически во время сборки)
├── BUILDROOT/                      # (Виртуальный корень системы во время сборки)
├── RPMS/                           # Сюда rpmbuild положит ГОТОВЫЕ .rpm пакеты!
│   └── noarch/                     
│       └── tz-cli-1.0-1.el8.noarch.rpm
│
├── SOURCES/                        # Сюда вы вручную кладете исходные архивы проектов
│   └── tz-cli-1.0.tar.gz
│
├── SPECS/                          # Сюда вы вручную кладете рецепты сборки (.spec файлы)
│   └── tz-cli.spec
│
└── SRPMS/                          # (Исходные RPM пакеты, создаются автоматически)
```