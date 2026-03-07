# Пример 2: Проект "погода" weather-cli, сборка через Setuptools

### Python-программа (Модульная структура, requests, сборка через Setuptools)

Для упаковки более сложных Python-программ (с несколькими файлами, модулями и сторонними библиотеками) используется стандартный механизм сборки Python — **setuptools** (через файл `setup.py` или `pyproject.toml`).

В этом примере мы создадим консольную утилиту `weather-cli`, которая:
1. Имеет модульную структуру (разделена на несколько файлов).
2. Зависит от сторонней библиотеки (`requests`).
3. Автоматически создает исполняемую команду в терминале (через `entry_points`).

Установим необходимые пакеты для сборки:
```bash
sudo dnf install python3-devel python3-setuptools
```

**Структура проекта перед архивацией (`tar -czvf`):**

```text
weather-cli-1.0/
├── setup.py
└── weather_cli/
    ├── __init__.py
    ├── core.py
    └── cli.py
```

---

### Шаг 1: Создание структуры проекта

Создадим директорию для исходного кода нашей программы:
```bash
mkdir -p ~/weather-cli-1.0/weather_cli
cd ~/weather-cli-1.0
```

#### 1. Файл `weather_cli/__init__.py`
Просто создайте пустой файл, чтобы Python понимал, что это пакет:
```bash
touch weather_cli/__init__.py
```

#### 2. Файл `weather_cli/core.py` (Основная логика)
Создайте файл и добавьте код для получения погоды (мы используем бесплатный сервис `wttr.in`, не требующий API-ключей):
```python
# weather_cli/core.py
import requests

def get_weather(city):
    try:
        url = f"https://wttr.in/{city}?format=3"
        response = requests.get(url, timeout=5)
        response.raise_for_status()
        return response.text.strip()
    except requests.exceptions.RequestException as e:
        return f"Ошибка при получении данных о погоде: {e}"
```

#### 3. Файл `weather_cli/cli.py` (Интерфейс командной строки)
Создайте файл для обработки аргументов терминала:
```python
# weather_cli/cli.py
import sys
import argparse
from .core import get_weather

def main():
    parser = argparse.ArgumentParser(description="Узнать погоду в городе.")
    parser.add_argument("city", nargs="?", default="Moscow", help="Название города (на английском)")
    args = parser.parse_args()

    print(f"Запрашиваем погоду для: {args.city}...")
    weather = get_weather(args.city)
    print(weather)

if __name__ == "__main__":
    main()
```

#### 4. Файл `setup.py` (Инструкция для сборки Python)
Создайте файл в корне `~/weather-cli-1.0/`:
```python
# setup.py
from setuptools import setup, find_packages

setup(
    name="weather-cli",
    version="1.0",
    packages=find_packages(),
    install_requires=[
        "requests", # Указываем зависимость
    ],
    entry_points={
        "console_scripts": [
            # Формат: команда_в_терминале = пакет.модуль:функция
            "weather=weather_cli.cli:main", 
        ]
    }
)
```

---

### Шаг 2: Упаковка исходников в архив

Выходим на уровень выше, создаем tar-архив и копируем его в папку RPM-сборщика:
```bash
cd ~
tar -czvf weather-cli-1.0.tar.gz weather-cli-1.0/
mv weather-cli-1.0.tar.gz ~/rpmbuild/SOURCES/
```

---

### Шаг 3: Написание SPEC-файла (Магия CentOS)

При упаковке сложных Python-программ в RPM есть **важное правило**:
Мы не используем `pip install` внутри RPM. Мы должны собирать программу системными утилитами и использовать **системные зависимости** (RPM-пакеты). Зависимость Python `requests` в CentOS называется `python3-requests`.

Создадим SPEC-файл:
```bash
nano ~/rpmbuild/SPECS/weather-cli.spec
```

Скопируйте в него следующее содержимое:

```specfile
Name:           weather-cli
Version:        1.0
Release:        1%{?dist}
Summary:        A command line weather tool using Python
License:        MIT
Source0:        %{name}-%{version}.tar.gz
BuildArch:      noarch

# Зависимости ДЛЯ СБОРКИ пакета (sudo dnf install python3-devel python3-setuptools)
BuildRequires:  python3-devel
BuildRequires:  python3-setuptools

# Зависимости ДЛЯ РАБОТЫ программы (устанавливать не нужно, будет скачано автоматически при установке собранного пакета)
Requires:       python3
Requires:       python3-requests

%description
Weather CLI is a modular Python application that fetches 
current weather data from wttr.in. It demonstrates how to 
package setuptools-based Python projects into RPMs.

%prep
%setup -q

%build
# Системный макрос CentOS для вызова: python3 setup.py build
%py3_build

%install
# Системный макрос CentOS для вызова: python3 setup.py install --root %{buildroot}
%py3_install

%files
# 1. Бинарный файл-скрипт (созданный через entry_points в setup.py)
%{_bindir}/weather

# 2. Директория с нашим исходным кодом в системной папке Python
%{python3_sitelib}/weather_cli/

# 3. Метаданные пакета (информация о версии и зависимостях)
%{python3_sitelib}/weather_cli-*.egg-info/

%changelog
* Fri Mar 06 2026 Admin - 1.0-1
- Initial release of weather-cli using setuptools.
```

**Особенности этого SPEC-файла:**
*   `BuildRequires: python3-setuptools` — говорит сборщику, что для выполнения секций `%build` и `%install` нужна библиотека setuptools.
*   `%py3_build` и `%py3_install` — это встроенные макросы RPM, которые автоматически делают всю сложную работу по правильной компиляции Python-кода (создание файлов `.pyc`) и раскладыванию файлов по нужным системным папкам (`/usr/lib/python3.x/site-packages/`).
*   `%{python3_sitelib}` — переменная, которая автоматически указывает на правильную системную директорию библиотек Python (например, `/usr/lib/python3.6/site-packages/` для CentOS 8).

---

### Шаг 4: Сборка пакета

Запустите сборку:
```bash
rpmbuild -ba ~/rpmbuild/SPECS/weather-cli.spec
```

Во время сборки вы увидите, как `setuptools` собирает пакет, генерирует файл-обертку для `entry_points` и копирует всё в виртуальный `buildroot`.
В конце должно быть `+ exit 0`.

Проверим наличие пакета:
```bash
ls -l ~/rpmbuild/RPMS/noarch/weather-cli-1.0-1.*.rpm
```

---

### Шаг 5: Установка и Тестирование

Установим наш пакет через `dnf`. Обратите внимание, что `dnf` **сам скачает и установит** зависимость `python3-requests`, так как мы указали её в `Requires`!

```bash
sudo dnf localinstall ~/rpmbuild/RPMS/noarch/weather-cli-1.0-1.*.rpm -y
```

После установки `setup.py` (через RPM) создал команду `weather`. Давайте протестируем:

```bash
weather
```
*Вывод:*
```text
Запрашиваем погоду для: Moscow...
Moscow: ⛅️  +5°C
```

Или с аргументом:
```bash
weather London
```
*Вывод:*
```text
Запрашиваем погоду для: London...
London: 🌧  +12°C
```

Поздравляю! Вы успешно собрали RPM пакет из модульного Python-проекта.

---

### Что делать, если сторонней библиотеки нет в репозиториях CentOS?
В нашем примере `python3-requests` есть в стандартных репозиториях. Но если ваша программа использует специфическую библиотеку (например, `fastapi`), которой нет в репозиториях ОС (или EPEL), у вас есть 3 пути:
1. **Правильный путь:** Упаковать саму недостающую библиотеку в отдельный RPM-пакет (создать spec-файл для `fastapi`), а затем требовать её.
2. **Путь "Виртуального окружения" (Virtualenv) (3-ий файл):** Создать RPM, который при установке помещает программу в `/opt/my_app/`, разворачивает там `venv` и скачивает всё через `pip` на этапе сборки. (Так делают для больших монолитов).
3. **Использовать `pex` или `PyInstaller`:** Скомпилировать весь Python код вместе с интерпретатором и зависимостями в один бинарный файл, и упаковать его как в *простом примере* из предыдущего ответа.