# Пример 5: Проект "местное время" rich-tz, сборка через Virtualenv (venv) 

### Python-программа (Virtualenv + библиотека форматированного вывода Rich)

Чтобы оправдать использование Virtualenv, мы добавим стороннюю библиотеку `rich`, которой нет в базовой CentOS. Она сделает вывод в терминале красивым (с таблицами и цветами). Скрипт по-прежнему использует команды Linux для получения данных.

Здесь мы добавляем файл зависимостей, так как нам нужно, чтобы при сборке RPM-пакета утилита `pip` скачала стороннюю библиотеку `rich` из интернета и положила её внутрь нашего пакета. Логику для простоты мы объединили в один файл.

**Структура исходного кода перед архивацией (`tar -czvf`):**

```text
~/rich-tz-1.0/                      # Корневая папка проекта (версия 1.0)
│
├── requirements.txt                # Список внешних зависимостей (в нашем случае: rich==13.6.0)
│
├── setup.py                        # Инструкция для setuptools (создание команды rich-tz)
│
└── rich_tz/                        # Главная папка Python-пакета
    │
    ├── __init__.py                 # Пустой маркер модуля
    │
    └── main.py                     # Весь код программы: получение времени и красивая отрисовка
```

---

#### Шаг 1. Создание проекта

```bash
mkdir -p ~/rich-tz-1.0/rich_tz
cd ~/rich-tz-1.0
touch rich_tz/__init__.py
```

**1. Файл `requirements.txt`**
```text
rich==13.6.0
```

**2. Основной код `rich_tz/main.py`**
```python
# rich_tz/main.py
import sys
import subprocess
import os
from rich.console import Console
from rich.panel import Panel

console = Console()

def get_valid_timezones():
    result = subprocess.run(['timedatectl', 'list-timezones'], capture_output=True, text=True)
    if result.returncode != 0:
        return []
    return result.stdout.splitlines()

def get_time(tz_name):
    env = os.environ.copy()
    env['TZ'] = tz_name
    result = subprocess.run(['date', '+%Y-%m-%d %H:%M:%S'], env=env, capture_output=True, text=True)
    return result.stdout.strip()

def main():
    tz_target = sys.argv[1] if len(sys.argv) > 1 else "UTC"
    valid_zones = get_valid_timezones()

    if valid_zones and tz_target not in valid_zones:
        console.print(f"[bold red]Ошибка:[/bold red] Временная зона '{tz_target}' не существует.")
        sys.exit(1)

    time_str = get_time(tz_target)
    
    # Красивый вывод через библиотеку rich
    console.print(Panel.fit(
        f"[bold cyan]Временная зона:[/bold cyan] {tz_target}\n[bold yellow]Локальное время:[/bold yellow] {time_str}",
        title="🕒 Системное время",
        border_style="blue"
    ))

if __name__ == "__main__":
    main()
```

**3. Файл сборки `setup.py`**
```python
# setup.py
from setuptools import setup, find_packages

setup(
    name="rich-tz",
    version="1.0",
    packages=find_packages(),
    entry_points={
        "console_scripts": [
            "rich-tz=rich_tz.main:main",
        ]
    }
)
```

#### Шаг 2. Создание архива и SPEC-файла (Магия VENV)

```bash
cd ~
tar -czvf rich-tz-1.0.tar.gz rich-tz-1.0/
mv rich-tz-1.0.tar.gz ~/rpmbuild/SOURCES/
nano ~/rpmbuild/SPECS/rich-tz.spec
```

**Содержимое `rich-tz.spec`:**
*(Обратите внимание: мы по-прежнему вырезаем пути сборки через `sed`, чтобы приложение работало в `/opt`)*
```specfile
Name:           rich-tz
Version:        1.0
Release:        1%{?dist}
Summary:        Timezone CLI tool packed with virtualenv
License:        MIT
Source0:        %{name}-%{version}.tar.gz
# УДАЛЕНО: BuildArch: noarch

# Отключаем авто-поиск зависимостей RPM для внутренностей venv!
AutoReqProv:    no

BuildRequires:  python3
Requires:       python3
Requires:       systemd
Requires:       coreutils

# Отключаем создание debug-пакетов (для Python они не нужны)
%define debug_package %{nil}

# ОТКЛЮЧАЕМ СОЗДАНИЕ СИСТЕМНЫХ ССЫЛОК BUILD-ID
%define _build_id_links none

# Определяем путь установки в системе
%define app_dir /opt/%{name}

%description
A beautiful timezone CLI tool packed in its own virtual environment.

%prep
%setup -q

%build
# Пусто, вся магия в install

%install
mkdir -p %{buildroot}%{app_dir}

# 1. Создаем venv
python3 -m venv %{buildroot}%{app_dir}/venv

# 2. Ставим зависимости (тут скачается rich)
%{buildroot}%{app_dir}/venv/bin/pip install --upgrade pip setuptools
%{buildroot}%{app_dir}/venv/bin/pip install -r requirements.txt

# 3. Ставим наше приложение
%{buildroot}%{app_dir}/venv/bin/pip install .

# 4. ИСПРАВЛЯЕМ ПУТИ: удаляем следы сборочной директории
sed -i "s|%{buildroot}||g" %{buildroot}%{app_dir}/venv/bin/*
sed -i "s|%{buildroot}||g" %{buildroot}%{app_dir}/venv/pyvenv.cfg

# 5. Делаем команду доступной глобально (относительный симлинк)
mkdir -p %{buildroot}%{_bindir}
ln -s ../../opt/%{name}/venv/bin/rich-tz %{buildroot}%{_bindir}/rich-tz

%files
%{app_dir}/
%{_bindir}/rich-tz

%changelog
* Fri Mar 06 2026 Admin - 1.0-1
- Release with virtualenv and linux system tools
```

**Сборка и проверка:**
```bash
rpmbuild -ba ~/rpmbuild/SPECS/rich-tz.spec

ls -l ~/rpmbuild/RPMS/x86_64/
# или aarch64, если вы на ARM

sudo dnf localinstall ~/rpmbuild/RPMS/x86_64/rich-tz-1.0-1.*.rpm -y
```

**Тестируем:**
```bash
rich-tz Asia/Tokyo
```

*Вывод в терминале будет отрисован синими рамками и цветным текстом (благодаря `rich`), но сами данные приложение получило из недр Linux (от `date`):*
```text
╭───── Системное время ────────────────────╮
│ Временная зона: Asia/Tokyo               │
│ Локальное время: 2023-10-23 23:15:00     │
╰──────────────────────────────────────────╯
```

Если ввести несуществующую зону (например, `rich-tz Narnia`), скрипт вызовет `timedatectl`, не найдет там `Narnia` и выдаст красную ошибку.



---

### Структура директории в рабочем пространстве CentOS (`~/rpmbuild/`) для обоих проектов

После того как вы упаковали исходники в `.tar.gz` и написали `.spec` файлы, ваша стандартизированная сборочная директория `rpmbuild` должна выглядеть именно так:

```text
~/rpmbuild/
│
├── BUILD/                          # (Папка используется автоматически во время сборки)
├── BUILDROOT/                      # (Виртуальный корень системы во время сборки)
├── RPMS/                           # Сюда rpmbuild положит ГОТОВЫЕ .rpm пакеты!
│   └── noarch/                     
│       ├── tz-cli-1.0-1.el8.noarch.rpm
│       └── rich-tz-1.0-1.el8.noarch.rpm
│
├── SOURCES/                        # Сюда вы вручную кладете исходные архивы проектов
│   ├── tz-cli-1.0.tar.gz
│   └── rich-tz-1.0.tar.gz
│
├── SPECS/                          # Сюда вы вручную кладете рецепты сборки (.spec файлы)
│   ├── tz-cli.spec
│   └── rich-tz.spec
│
└── SRPMS/                          # (Исходные RPM пакеты, создаются автоматически)
```