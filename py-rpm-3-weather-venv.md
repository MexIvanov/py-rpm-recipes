# Пример 3: Проект "погода" rich-weather, сборка через Virtualenv (venv)

### Python-программа (Virtualenv + библиотека форматированного вывода Rich, requests)

Создание RPM-пакета, который внутри содержит собственное виртуальное окружение (Virtualenv) — это **стандарт для упаковки корпоративных (Enterprise) приложений на Python**.

**Зачем это нужно?**
1. Вы не зависите от системных RPM-пакетов библиотек (которых часто нужных версий нет в репозиториях CentOS).
2. Вы скачиваете любые пакеты через `pip` (например, `Django`, `FastAPI`, `Rich`) на этапе *сборки*.
3. Ваше приложение изолировано в папке `/opt/` и никак не сломает системный Python.

Мы создадим красивую утилиту `rich-weather`, использующую библиотеку `rich` (для цветного текста), которой по умолчанию нет в CentOS.

---

### Шаг 1: Подготовка проекта

Создадим структуру проекта:
```bash
mkdir -p ~/rich-weather-1.0/rich_weather
cd ~/rich-weather-1.0
```

#### 1. Код программы (`rich_weather/main.py`)
Мы используем `requests` для сети и `rich` для красивого вывода.
```python
# rich_weather/main.py
import sys
import requests
from rich.console import Console
from rich.panel import Panel

console = Console()

def main():
    city = sys.argv[1] if len(sys.argv) > 1 else "Moscow"
    url = f"https://wttr.in/{city}?format=3"
    
    try:
        response = requests.get(url, timeout=5)
        response.raise_for_status()
        weather = response.text.strip()
        
        # Красивый вывод с помощью библиотеки rich
        console.print(Panel.fit(
            f"[bold cyan]Город:[/bold cyan] {city}\n[bold yellow]Погода:[/bold yellow] {weather}",
            title="🌤 Сводка погоды",
            border_style="green"
        ))
    except Exception as e:
        console.print(f"[bold red]Ошибка:[/bold red] {e}")

if __name__ == "__main__":
    main()
```

#### 2. Обязательный файл `rich_weather/__init__.py`
```bash
touch rich_weather/__init__.py
```

#### 3. Файл зависимостей `requirements.txt`
Здесь мы указываем всё, что `pip` должен скачать при сборке.
```text
requests==2.31.0
rich==13.6.0
```

#### 4. Файл сборки `setup.py`
```python
# setup.py
from setuptools import setup, find_packages

setup(
    name="rich-weather",
    version="1.0",
    packages=find_packages(),
    entry_points={
        "console_scripts": [
            "rich-weather=rich_weather.main:main",
        ]
    }
)
```

---

### Шаг 2: Создание архива

Упаковываем всё в tar-архив и отправляем сборщику RPM:
```bash
cd ~
tar -czvf rich-weather-1.0.tar.gz rich-weather-1.0/
mv rich-weather-1.0.tar.gz ~/rpmbuild/SOURCES/
```

---

### Шаг 3: Написание SPEC-файла (Магия Virtualenv)

Сборка пакета с `venv` имеет **два главных нюанса**:
1. Изолированные программы принято ставить в `/opt/<имя_программы>`.
2. Когда мы создаем `venv` внутри временной папки сборки (`%{buildroot}`), Python жестко прописывает эти пути в файлах. Нам придется исправить (вырезать) пути сборки с помощью `sed`, чтобы на целевом сервере пути вели просто в `/opt/`.

Создайте SPEC-файл:
```bash
nano ~/rpmbuild/SPECS/rich-weather.spec
```

Скопируйте в него следующий код:

```specfile
Name:           rich-weather
Version:        1.0
Release:        1%{?dist}
Summary:        Weather CLI app packed with virtualenv
License:        MIT
Source0:        %{name}-%{version}.tar.gz
# УДАЛЕНО: BuildArch: noarch

# Отключаем автоматический поиск зависимостей RPM!
# Иначе RPM попытается сделать пакет зависимым от сотен библиотек,
# которые лежат внутри нашего venv, и пакет не установится.
AutoReqProv:    no

BuildRequires:  python3
Requires:       python3

# Отключаем создание debug-пакетов (для Python они не нужны)
%define debug_package %{nil}

# ОТКЛЮЧАЕМ СОЗДАНИЕ СИСТЕМНЫХ ССЫЛОК BUILD-ID (Решение вашей ошибки!)
%define _build_id_links none

# Определяем путь установки в системе
%define app_dir /opt/%{name}

%description
Rich Weather is a beautifully formatted CLI weather app.
It brings its own Python virtual environment, making it 
completely independent of system Python libraries.

%prep
%setup -q

%build
# В секции build для virtualenv обычно ничего не делают, 
# вся работа происходит в install

%install
# 1. Создаем папку программы в виртуальном корне сборки
mkdir -p %{buildroot}%{app_dir}

# 2. Создаем виртуальное окружение
python3 -m venv %{buildroot}%{app_dir}/venv

# 3. Обновляем pip внутри venv (хорошая практика)
%{buildroot}%{app_dir}/venv/bin/pip install --upgrade pip setuptools

# 4. Устанавливаем зависимости из requirements.txt через pip
%{buildroot}%{app_dir}/venv/bin/pip install -r requirements.txt

# 5. Устанавливаем само наше приложение внутрь venv
%{buildroot}%{app_dir}/venv/bin/pip install .

# 6. КРИТИЧЕСКИ ВАЖНЫЙ ШАГ: Исправляем пути (shebangs)
# Python при установке прописал в скриптах путь вида #!/root/rpmbuild/BUILDROOT/...
# Нам нужно заменить это на реальный путь: #!/opt/rich-weather/...
sed -i "s|%{buildroot}||g" %{buildroot}%{app_dir}/venv/bin/*
sed -i "s|%{buildroot}||g" %{buildroot}%{app_dir}/venv/pyvenv.cfg

# 7. Создаем символическую ссылку в /usr/bin, чтобы команду было удобно вызывать
mkdir -p %{buildroot}%{_bindir}
ln -s ../../opt/%{name}/venv/bin/rich-weather %{buildroot}%{_bindir}/rich-weather

%files
# Указываем, что RPM-пакет владеет всей папкой /opt/rich-weather и симлинком
%{app_dir}/
%{_bindir}/rich-weather

%changelog
* Fri Mar 06 2026 Admin - 1.0-1
- Initial release with embedded virtualenv
```

---

### Шаг 4: Сборка пакета

Убедитесь, что сервер, на котором вы собираете пакет, имеет доступ в интернет, так как `pip` будет скачивать пакеты (`requests`, `rich`).

Запустите сборку:
```bash
rpmbuild -ba ~/rpmbuild/SPECS/rich-weather.spec
```

*Во время сборки вы увидите логи `pip install`, скачивающего зависимости. Это нормально.*

Проверим размер пакета:
```bash
ls -lh ~/rpmbuild/RPMS/noarch/rich-weather-1.0-1.*.rpm
```
Вы заметите, что пакет весит несколько мегабайт. Это потому, что внутри него запакованы все библиотеки, сам Python-интерпретатор (ссылки на него) и весь `pip`. Это плата за полную автономность.

---

### Шаг 5: Установка и Тестирование пакета

Установим пакет. Теперь ему не нужен доступ в интернет — всё уже внутри RPM:

```bash
sudo dnf localinstall ~/rpmbuild/RPMS/noarch/rich-weather-1.0-1.*.rpm -y
```

Давайте посмотрим, как программа разместилась в системе:
```bash
ls -l /opt/rich-weather/
```
Там лежит папка `venv`.

Запустим программу! Благодаря симлинку в `/usr/bin/rich-weather`, она доступна глобально:

```bash
rich-weather Moscow
```
*Вывод будет в красивой зеленой рамочке (благодаря библиотеке `rich` из виртуального окружения):*
```text
╭─ 🌤 Сводка погоды ──────────────────────╮
│ Город: Moscow                            │
│ Погода: Moscow: ☁️  +14°C                │
╰──────────────────────────────────────────╯
```

---

### Резюме этого подхода (Virtualenv в RPM)

**Плюсы:**
* **Полная изоляция:** Вы можете использовать `Django 4.0` в своей программе, даже если в CentOS глобально установлен `Django 1.11`.
* **Надежность:** Приложение никогда не сломается после случайного `yum update`, который может обновить системные Python-библиотеки.
* **Простота разработки:** Вы можете использовать любые пакеты из PyPI (pip), не дожидаясь, пока их переведут в формат RPM для CentOS.

**Минусы:**
* **Размер пакета:** Если у вас 10 таких микросервисов, каждый принесет с собой свою копию библиотек. Увеличивается расход диска.
* **Безопасность (CVE):** Если в библиотеке `requests` найдут уязвимость, вам придется пересобрать *ваш RPM-пакет*. Обновление системы командой `dnf upgrade` не обновит библиотеки внутри вашей папки `/opt/.../venv/`.