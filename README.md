# mirroring-template

Набор GitLab CI/CD шаблонов для зеркалирования артефактов между разными источниками.

---

## oci-repository

Копирование между oci репозиториями с поддержкой идемпотентности.

### 🔐 Авторизация в Registry (Инструкция для разработчиков)

> ⚠️ **Критически важно:** Ни при каких обстоятельствах не пишите логины и пароли текстом в файлы `.yml`!

Все доступы к приватным реестрам передаются через **Settings -> CI/CD -> Variables** в интерфейсе GitLab (с обязательным флагом **Mask variable**) либо внутри блока `parallel:matrix` в вашем проекте.

### Имена переменных для авторизации:

1. **Для источника (Откуда качаем образ):**
  * `SRC_REGISTRY_USER` — логин.
  * `SRC_REGISTRY_PASSWORD` — пароль или токен.
  * _Если образ публичный (например, на Docker Hub), эти переменные создавать **не нужно**._

2. **Для назначения (Куда пушим образ):**
  * `DEST_REGISTRY_USER` — логин.
  * `DEST_REGISTRY_PASSWORD` — пароль или токен.
  * ℹ️ **Поведение по умолчанию:** Если вы оставите эти переменные **пустыми**, шаблон автоматически подставит системный токен текущего пайплайна (`$CI_JOB_TOKEN`). Это значит, что для загрузки образов во встроенный Container Registry вашего же проекта ничего настраивать не нужно — всё заведется само.


### Usage

```yaml
include:
  - component: $CI_SERVER_FQDN/library/cicd/templates/mirroring/oci-repository@main
    inputs:
      stage: "test"
      src-oci-image: "$EXAMPLE_ENV_SRC"
      dest-oci-image: "$EXAMPLE_ENV_DEST"

variables:
  EXAMPLE_ENV_SRC: "docker.io/library/hello-world"
  EXAMPLE_ENV_DEST: "registry.gitlab.fizn.ru/library/cicd/templates/mirroring/oci-repository"

stages:
  - test


```

### inputs

| Параметр                   | Описание                                      | Тип     | По умолчанию                           | Обязательный |
|----------------------------|-----------------------------------------------|---------|----------------------------------------|--------------|
| `mirroring-oci-repository` | Enable OCI repository mirroring               | boolean | true                                   | Нет          |
| `job-prefix`               | Job prefix                                    | string  | `-$CI_PROJECT_ID-$CI_COMMIT_SHORT_SHA` | Нет          |
| `skopeo-image`             | Skopeo CLI docker image                       | string  | quay.io/containers/skopeo:latest       | Нет          |
| `stage`                    | Job stage                                     | string  | publish                                | Нет          |
| `skopeo-args`              | Extra args for skopeo                         | string  | ""                                     | Нет          |
| `src-oci-image`            | Source image (supports variables/matrix)      | string  | ""                                     | Нет          |
| `dest-oci-image`           | Destination image (supports variables/matrix) | string  | ""                                     | Нет          |

### Environment Variables

| Переменная окружения | Значение                       | Описание / Поведение в рантайме                                                                                      |
|----------------------|--------------------------------|----------------------------------------------------------------------------------------------------------------------|
| `SKOPEO_IMAGE`       | `$[[ inputs.skopeo-image ]]`   | Докер-образ для запуска утилиты Skopeo.                                                                              |
| `SKOPEO_OCI_ARGS`    | `$[[ inputs.skopeo-args ]]`    | Дополнительные аргументы командной строки для команд Skopeo.                                                         |
| `SRC_RELEASE_IMAGE`  | `$[[ inputs.src-oci-image ]]`  | Умный фолбэк: берет значение из matrix или внешних переменных. Если они пусты, использует src-oci-image из инпутов.  |
| `DEST_RELEASE_IMAGE` | `$[[ inputs.dest-oci-image ]]` | Умный фолбэк: берет значение из matrix или внешних переменных. Если они пусты, использует dest-oci-image из инпутов. |

### Template Extension Points

| Расширение / Блок                                  | Назначение                                      |
|----------------------------------------------------|-------------------------------------------------|
| `.mirroring:oci-repository:public:env-overwrite`   | Переопределение переменных окружения.           |
| `.mirroring:oci-repository:public:rules-extends`   | Переопределение правил запуска.                 |
| `.mirroring:oci-repository:public:extends`         | Переопределение блоков mirroring job.           |
| `.mirroring:oci-repository:public:needs`           | Определяет зависимости mirroring от других job. |
| `.mirroring:mirroring-core:shared:tbc_common`      | Общие скрипты **TBC**                           |
| `.mirroring:mirroring-core:shared:registries_auth` | Генерация `~/.docker/config.json`               |

---

## git-repository
Копирование между Git-репозиториями с поддержкой синхронизации как отдельных веток, так и полного зеркалирования всех ссылок (branches/tags).
### 🔐 Авторизация в Git-репозиториях через .netrc

⚠️ Критически важно: Ни при каких обстоятельствах не пишите логины, пароли или токены текстом в файлы `.yml`! Не передавайте секреты открытым текстом внутри URL-адресов репозиториев.

Авторизация в приватных Git-репозиториях полностью автоматизирована и изолирована с помощью механизма `~/.netrc` и встроенного шаблонизатора переменных `tbc_envsubst`.
Шаблон поддерживает два сценария авторизации:
#### Сценарий 1: Автоматический (Встроенный GitLab) — По умолчанию
Если в корне вашего проекта отсутствует файл `.netrc`, шаблон автоматически сконфигурирует авторизацию для хоста текущего GitLab-сервера (`$CI_SERVER_HOST`).

* В качестве логина используется: `gitlab-ci-token`
* В качестве пароля используется: системный `$CI_JOB_TOKEN` текущего пайплайна.
* Что это значит: Для скачивания или пуша в репозитории внутри вашего же инстанса GitLab ничего настраивать не нужно — всё заработает «из коробки».

#### Сценарий 2: Кастомный (Внешние репозитории: GitHub, Bitbucket и др.)
Если вам нужно авторизоваться на внешних Git-хостингах или использовать кастомные токены, создайте в корне своего проекта файл .netrc с переменными-плейсхолдерами.
1. Создайте файл `.netrc` в корне проекта:
    ```
    machine github.com
    login ${GITHUB_USER}
    password ${GITHUB_TOKEN}
    
    machine example.com
    login oauth2
    password ${INTERNAL_GITLAB_TOKEN}
    ```

2. Задайте секреты в интерфейсе GitLab:
   Перейдите в **Settings** -> **CI/CD** -> **Variables** и добавьте переменные **GITHUB_USER**, **GITHUB_TOKEN** и **INTERNAL_GITLAB_TOKEN** (обязательно включите для них флаг Mask variable).
3. Как это работает в рантайме:
   Встроенная функция `tbc_envsubst` перед стартом Git-команд автоматически подставит значения из маскированных переменных GitLab CI в ваш файл `.netrc`, переместит его в безопасную домашнюю директорию (`~/.netrc`) и выставит правильные права доступа 0600.  
   При этом механизм `tbc_envsubst` автоматически экранирует спецсимволы в паролях и токенах, предотвращая синтаксические ошибки при работе с URL.

------------------------------
### Usage
При одиночном запуске параметры передаются стандартным образом через inputs.
```yaml
include:
  - component: $CI_SERVER_FQDN/library/cicd/templates/mirroring/git-repository@main
    inputs:
      stage: "test"
      src-git-repository: "$EXAMPLE_ENV_SRC"
      dest-git-repository: "$EXAMPLE_ENV_DEST"
      branch: "master"
variables:
  EXAMPLE_ENV_SRC: "https://github.com"
  EXAMPLE_ENV_DEST: "https://oauth2:$MY_GITLAB_TOKEN@://company.com"
stages:
  - test
```

### inputs

| Параметр                 | Описание                                                             | Тип     | По умолчанию                           | Обязательный |
|--------------------------|----------------------------------------------------------------------|---------|----------------------------------------|--------------|
| mirroring-git-repository | Enable GIT repository mirroring                                      | boolean | `true`                                 | Нет          |
| job-prefix               | Job prefix                                                           | string  | `-$CI_PROJECT_ID-$CI_COMMIT_SHORT_SHA` | Нет          |
| git-image                | Git CLI docker image                                                 | string  | `alpine/git:latest`                    | Нет          |
| stage                    | Job stage                                                            | string  | `publish`                              | Нет          |
| git-args                 | Extra args for git push (e.g. --atomic)                              | string  | ""                                     | Нет          |
| src-git-repository       | Source repository URL (supports variables/matrix)                    | string  | ""                                     | Нет          |
| dest-git-repository      | Destination repository URL (supports variables/matrix)               | string  | ""                                     | Нет          |
| branch                   | Specific branch to mirror. If left empty, mirrors all branches/tags. | string  | ""                                     | Нет          |

------------------------------
### Environment Variables

| Переменная окружения | Значение                            | Описание / Поведение в рантайме                                                                                             |
|----------------------|-------------------------------------|-----------------------------------------------------------------------------------------------------------------------------|
| GIT_IMAGE            | `$[[ inputs.git-image ]]`           | Докер-образ для запуска Git.                                                                                                |
| GIT_ARGS             | `$[[ inputs.git-args ]]`            | Дополнительные аргументы командной строки для git push.                                                                     |
| SRC_GIT_REPOSITORY   | `$[[ inputs.src-git-repository ]]`  | Умный фолбэк: приоритетно берет значение из matrix или внешних переменных. Если они пусты, берет инпут src-git-repository.  |
| DEST_GIT_REPOSITORY  | `$[[ inputs.dest-git-repository ]]` | Умный фолбэк: приоритетно берет значение из matrix или внешних переменных. Если они пусты, берет инпут dest-git-repository. |
| GIT_BRANCH           | `$[[ inputs.branch ]]`              | Умный фолбэк: приоритетно берет ветку из matrix. Если пустая — зеркалирует весь репозиторий (--mirror).                     |

------------------------------
### Template Extension Points
Вы можете гибко управлять поведением джобы из своего проекта, переопределяя следующие скрытые блоки:

| Расширение / Блок                              | Назначение                                                                                  |
|------------------------------------------------|---------------------------------------------------------------------------------------------|
| .mirroring:git-repository:public:env-overwrite | Переопределение рантайм-переменных окружения и логики фолбэков.                             |
| .mirroring:git-repository:public:rules-extends | Переопределение условий запуска (по умолчанию: on_success при наличии Git-тегов).           |
| .mirroring:git-repository:public:extends       | Главная точка расширения. Сюда инжектится parallel:matrix, tags или кастомные variables.    |
| .mirroring:git-repository:public:needs         | Определяет зависимости текущей джобы от других этапов (по умолчанию список пуст).           |
| .mirroring:git-repository:internal:scripts     | Внутренняя инициализация (создание директорий, настройка дефолтных user.name и user.email). |
| .mirroring:mirroring-core:shared:tbc_common    | Интеграция общих скриптов и хелперов логирования TBC.                                       |
| .mirroring:mirroring-core:shared:netrc         | Автоматическая генерация файлов аутентификации .netrc (если применимо).                     |
