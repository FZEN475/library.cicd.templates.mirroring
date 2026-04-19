# mirroring-template

Набор GitLab CI/CD шаблонов для зеркалирования артефактов между разными источниками.

---

## oci-repository

Копирование между oci репозиториями с поддержкой идемпотентности.

### Usage

```yaml
include:
  - component: $CI_SERVER_FQDN/library/cicd/templates/mirroring/oci-repository@main
    inputs:
      stage: "test"
      src-auth-file: "/tmp/config.json"
      dest-auth-file: "/tmp/config.json"
      src-oci-image: "$EXAMPLE_ENV_SRC"
      dest-oci-image: "$EXAMPLE_ENV_DEST"

variables:
  EXAMPLE_ENV_SRC: "stc/image"
  EXAMPLE_ENV_DEST: "dect/image"

stages:
  - test

.mirroring:oci-repository:public:rules-extends:
  tags:
    - docker-builder

```

### inputs

| Параметр         | Описание                               | Тип       | По умолчанию                           | Обязательный |
|------------------|----------------------------------------|-----------|----------------------------------------|--------------|
| `oci-repository` | Enable oci repository                  | `boolean` | `true`                                 | Нет          |
| `job-prefix`     | Job prefix                             | `string`  | `-$CI_PROJECT_ID-$CI_COMMIT_SHORT_SHA` | Нет          |
| `skopeo-image`   | cosign version to download             | `string`  | `quay.io/containers/skopeo:latest`     | Нет          |
| `stage`          | Job stage                              | `string`  | `publish`                              | Нет          |
| `src-auth-file`  | Source auth file in docker format      | `string`  | `~/.docker/config.json`                | Нет          |
| `dest-auth-file` | Destination auth file in docker format | `string`  | `~/.docker/config.json`                | Нет          |
| `skopeo-args`    | Extra args for skopeo                  | `string`  | `""`                                   | Нет          |
| `src-oci-image`  | Source image (support env)             | `string`  | —                                      | Да           |
| `dest-oci-image` | Destination image (support env)        | `string`  | —                                      | Да           |

### Environment Variables

| Переменная окружения | Значение                       | Notes |
|----------------------|--------------------------------|-------|
| `SKOPEO_IMAGE`       | `$[[ inputs.skopeo-image ]]`   |       |
| `SRC_OCI_IMAGE`      | `$[[ inputs.src-oci-image ]]`  |       |
| `DEST_OCI_IMAGE`     | `$[[ inputs.dest-oci-image ]]` |       |
| `SRC_AUTH_FILE`      | `$[[ inputs.src-auth-file ]]`  |       |
| `DEST_AUTH_FILE`     | `$[[ inputs.dest-auth-file ]]` |       |
| `SKOPEO_OCI_ARGS`    | `$[[ inputs.skopeo-args ]]`    |       |
| `SRC_OCI_IMAGE`      | `$[[ inputs.src-oci-image ]]`  |       |
| `DEST_OCI_IMAGE`     | `$[[ inputs.dest-oci-image ]]` |       |

### Template Extension Points

| Расширение / Блок                                | Назначение                                      |
|--------------------------------------------------|-------------------------------------------------|
| `.mirroring:oci-repository:public:env-overwrite` | Переопределение переменных окружения.           |
| `.mirroring:oci-repository:public:rules-extends` | Переопределение правил запуска.                 |
| `.mirroring:oci-repository:public:extends`       | Переопределение блоков mirroring job.           |
| `.mirroring:oci-repository:public:needs`         | Определяет зависимости mirroring от других job. |


