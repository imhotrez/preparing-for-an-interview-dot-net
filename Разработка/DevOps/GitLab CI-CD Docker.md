# Настройка CI/CD пайплайна в GitLab для ASP.NET Core приложения с развертыванием на Docker-сервере

В этой статье мы рассмотрим, как настроить полный цикл CI/CD для вашего ASP.NET Core приложения, размещенного в репозитории GitLab. Мы настроим автоматическое тестирование, сборку и развертывание приложения на Linux-сервере с Docker.

## Предпосылки

- **ASP.NET Core приложение**, размещенное в **GitLab репозитории**.
- **Linux-сервер** с установленным **Docker**, доступный по SSH.
- **GitLab Runner** (может быть использован общий Runner GitLab или настроен собственный).
- **Базовое понимание** Docker, GitLab CI/CD и SSH.

## Общая схема процесса

1. **GitLab CI/CD** будет использоваться для автоматизации процесса тестирования, сборки и развертывания.
2. **Docker** будет использоваться для контейнеризации приложения.
3. Собранный Docker-образ будет отправлен в **GitLab Container Registry**.
4. На удаленном сервере Docker будет запускать контейнер на основе последнего образа из реестра.

## Шаг 1: Настройка GitLab Container Registry

GitLab предоставляет встроенный контейнерный реестр, который мы будем использовать для хранения наших Docker-образов.

1. **Включите Container Registry** в настройках вашего проекта (обычно включен по умолчанию).
2. **Получите URL реестра**. Он будет выглядеть примерно так:

   ```
   registry.gitlab.com/your-namespace/your-project
   ```

## Шаг 2: Создание Dockerfile для вашего приложения

В корне вашего проекта создайте файл `Dockerfile`, который будет использоваться для сборки Docker-образа.

```dockerfile
# Используем официальный образ .NET Core SDK для сборки приложения
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /app

# Копируем файлы проекта и восстанавливаем зависимости
COPY *.csproj ./
RUN dotnet restore

# Копируем остальные файлы и собираем приложение
COPY . ./
RUN dotnet publish -c Release -o out

# Используем официальный образ .NET Core Runtime для запуска приложения
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS runtime
WORKDIR /app
COPY --from=build /app/out ./

# Указываем порт, который будет использоваться
EXPOSE 80

# Запускаем приложение
ENTRYPOINT ["dotnet", "YourApp.dll"]
```

Замените `YourApp.dll` на название вашего собранного DLL-файла.

## Шаг 3: Настройка `.gitlab-ci.yml`

Создайте файл `.gitlab-ci.yml` в корне вашего репозитория. Этот файл определяет пайплайн CI/CD.

```yaml
image: docker:latest

services:
  - docker:dind

variables:
  DOCKER_HOST: tcp://docker:2375/
  DOCKER_DRIVER: overlay2
  # Переменные для авторизации в Container Registry
  CI_REGISTRY: registry.gitlab.com
  CI_REGISTRY_IMAGE: $CI_REGISTRY/$CI_PROJECT_PATH

stages:
  - test
  - build
  - deploy

before_script:
  - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY

test:
  stage: test
  image: mcr.microsoft.com/dotnet/sdk:8.0
  script:
    - dotnet restore
    - dotnet test

build:
  stage: build
  script:
    - docker build -t $CI_REGISTRY_IMAGE:latest .
    - docker push $CI_REGISTRY_IMAGE:latest

deploy:
  stage: deploy
  script:
    - ssh root@your.server.ip "docker pull $CI_REGISTRY_IMAGE:latest && docker stop your_container_name || true && docker rm your_container_name || true && docker run -d --name your_container_name -p 80:80 $CI_REGISTRY_IMAGE:latest"
  only:
    - main
```

### Пояснение:

- **image**: Указываем, что используем Docker для выполнения команд.
- **services**: Запускаем сервис `docker:dind` (Docker-in-Docker), чтобы иметь возможность использовать Docker внутри контейнера Runner'а.
- **variables**: Указываем переменные среды для Docker и GitLab Registry.
- **stages**: Определяем три этапа: `test`, `build`, `deploy`.
- **before_script**: Входим в реестр Docker перед выполнением скриптов.
- **test**:
  - Используем образ SDK .NET для тестирования.
  - Выполняем команды `dotnet restore` и `dotnet test`.
- **build**:
  - Сборка Docker-образа и отправка его в GitLab Container Registry.
- **deploy**:
  - Подключаемся по SSH к серверу.
  - Выполняем команды для обновления контейнера с новым образом.
  - `only: - main` означает, что деплой будет происходить только при пуше в ветку `main`.

## Шаг 4: Настройка SSH-доступа для деплоя

Для того чтобы GitLab Runner мог подключиться по SSH к вашему серверу, необходимо настроить ключи SSH.

### Генерация SSH-ключа

На машине, где работает GitLab Runner, сгенерируйте SSH-ключ (если еще нет):

```bash
ssh-keygen -t rsa -b 4096 -C "gitlab@ci"
```

Ключ будет сохранен по пути `~/.ssh/id_rsa`. Публичный ключ будет в файле `id_rsa.pub`.

### Добавление публичного ключа на сервер

Скопируйте содержимое `id_rsa.pub` и добавьте его в файл `~/.ssh/authorized_keys` на вашем сервере:

```bash
cat id_rsa.pub >> ~/.ssh/authorized_keys
```

### Добавление приватного ключа в переменные GitLab CI/CD

В настройках вашего проекта в GitLab перейдите в **Settings > CI/CD > Variables** и добавьте переменную:

- **Key**: `SSH_PRIVATE_KEY`
- **Value**: Содержимое файла `id_rsa` (приватный ключ)
- **Type**: Variable
- **Flags**: Protect variable (если вы деплоите только из защищенных веток)

### Добавление известного хоста

Чтобы избежать подтверждения при первом подключении по SSH, добавьте отпечаток сервера в `known_hosts`.

Получите отпечаток сервера:

```bash
ssh-keyscan your.server.ip
```

Скопируйте вывод и добавьте его в переменную GitLab:

- **Key**: `SSH_KNOWN_HOSTS`
- **Value**: Вывод команды выше

### Обновление `.gitlab-ci.yml` для использования SSH

Измените секцию `deploy`:

```yaml
deploy:
  stage: deploy
  before_script:
    - 'which ssh-agent || ( apk add --update openssh-client )'
    - eval $(ssh-agent -s)
    - mkdir -p ~/.ssh
    - echo "$SSH_PRIVATE_KEY" | tr -d '\r' | ssh-add -
    - echo "$SSH_KNOWN_HOSTS" > ~/.ssh/known_hosts
  script:
    - ssh root@your.server.ip "docker pull $CI_REGISTRY_IMAGE:latest && docker stop your_container_name || true && docker rm your_container_name || true && docker run -d --name your_container_name -p 80:80 $CI_REGISTRY_IMAGE:latest"
  only:
    - main
```

### Пояснение:

- **before_script**:
  - Устанавливаем `openssh-client` при необходимости.
  - Запускаем `ssh-agent` и добавляем приватный ключ.
  - Настраиваем `known_hosts` для избежания подтверждения сервера.
- **script**:
  - Подключаемся по SSH и выполняем команды для обновления Docker-контейнера.

## Шаг 5: Настройка GitLab Runner (если требуется)

Если у вас нет своего Runner'а, и вы хотите использовать общий Runner GitLab, то этот шаг можно пропустить. Если же вы хотите настроить свой Runner, выполните следующие действия:

### Установка GitLab Runner на вашем сервере

Следуйте официальной [инструкции по установке GitLab Runner](https://docs.gitlab.com/runner/install/linux-repository.html) для вашей ОС.

### Регистрация Runner'а

После установки зарегистрируйте Runner:

```bash
sudo gitlab-runner register
```

- **URL GitLab**: `https://gitlab.com/` или ваш собственный сервер GitLab.
- **Token**: Получите токен в настройках вашего проекта **Settings > CI/CD > Runners**.
- **Description**: Любое описание.
- **Tags**: Можно оставить пустым или добавить теги.
- **Executor**: Выберите `docker`.
- **Image**: Укажите `docker:latest`.

### Настройка доступа к Docker

Убедитесь, что Runner имеет доступ к Docker сокету. Для этого можно использовать `docker:dind` (Docker-in-Docker) или смонтировать сокет хоста.

## Шаг 6: Настройка сервера для развертывания

Убедитесь, что на вашем сервере установлены Docker и необходимые порты открыты.

### Установка Docker

Следуйте официальной [инструкции по установке Docker](https://docs.docker.com/engine/install/) для вашей ОС.

### Проверка Docker

Убедитесь, что Docker работает корректно:

```bash
sudo docker run hello-world
```

## Шаг 7: Тестирование пайплайна

Теперь, когда все настроено, сделайте коммит и пуш в ветку `main`. Это должно запустить пайплайн CI/CD.

- Зайдите в ваш проект в GitLab и перейдите в **CI/CD > Pipelines**.
- Наблюдайте за выполнением пайплайна.
- Если все настроено правильно, после завершения пайплайна ваше приложение будет развернуто на сервере.

## Шаг 8: Проверка развернутого приложения

Перейдите по адресу вашего сервера в браузере:

```
http://your.server.ip/
```

Ваше приложение должно быть доступно.

## Дополнительные настройки и улучшения

### Использование переменных среды

Чтобы не хардкодить значения в `.gitlab-ci.yml`, используйте переменные среды.

Пример:

```yaml
variables:
  DOCKER_IMAGE_NAME: 'your_container_name'
  DEPLOY_SERVER: 'root@your.server.ip'
```

И замените эти значения в скрипте:

```yaml
script:
  - ssh $DEPLOY_SERVER "docker pull $CI_REGISTRY_IMAGE:latest && docker stop $DOCKER_IMAGE_NAME || true && docker rm $DOCKER_IMAGE_NAME || true && docker run -d --name $DOCKER_IMAGE_NAME -p 80:80 $CI_REGISTRY_IMAGE:latest"
```

### Обработка ошибок и откат

Добавьте обработку ошибок и возможность отката к предыдущей версии, если что-то пойдет не так.

### Безопасность

- **Защитите переменные CI/CD**: Убедитесь, что переменные с секретными данными защищены и не доступны для небезопасных веток.
- **Ограничьте доступ по SSH**: Используйте ограниченного пользователя вместо `root`, настройте брандмауэр и другие меры безопасности.

### Мониторинг и логирование

- Настройте мониторинг вашего приложения.
- Собирайте и анализируйте логи для быстрого обнаружения и устранения проблем.

## Заключение

Вы настроили полный цикл CI/CD для вашего ASP.NET Core приложения с использованием GitLab CI/CD и Docker. Теперь каждый раз при пуше в ветку `main` ваши изменения будут автоматически протестированы, собраны в Docker-образ, отправлены в реестр и развернуты на вашем сервере.

Эта настройка может быть расширена и улучшена в зависимости от ваших потребностей. Вы можете добавить больше этапов, настроить уведомления, интегрировать с другими сервисами и т.д.

**Примечание**: Всегда проверяйте безопасность вашей инфраструктуры и регулярно обновляйте используемые инструменты и пакеты.
