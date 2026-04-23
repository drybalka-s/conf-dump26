# Helm Deployment Component

## 🚀 Быстрый старт

### Минимальная конфигурация

```yaml
include:
  - component: $CI_SERVER_HOST/gitlab-components/helm/gitlab-ci-helm@v1
    inputs:
      helm_k8s_name: "prod"
      helm_release_name: "my-app"
      helm_release_namespace: "default"
      helm_diff_enabled: "true"  #включен по умолчанию

stages:
  - diff
  - deploy

```
---

## 📋 Параметры компонента

### Обязательные параметры ⭐

| Параметр | Тип | Описание | Пример |
|----------|-----|---------|--------|
| **`helm_release_name`** | string | Имя Helm release | `my-app` |
| **`helm_release_namespace`** | string | Имя namespace Helm release  | `default` |
| **`helm_k8s_name`** | string | Имя K8s кластера (преобразуется в KUBE_CONFIG_NAME) | `prod`, `staging`, `dev` |

**Важно:**
- все параметры ОБЯЗАТЕЛЬНЫ
- Без них job завершится с ошибкой


### Опциональные параметры

#### Путями и именами

| Параметр | Тип | Default | Описание |
|----------|-----|---------|---------|
| `helm_dir` | string | `.helm` | Путь к директории с Helm чартом |
| `helm_values_file` | string | `values.yaml` | Путь к файлу или файлам values `-f file1.yaml -f file2.yaml` |
| `helm_secrets_file` | string | `secrets.yaml` | Путь к файлу с secrets |
| `image_tag_prefix` | string | `image.tag` | Префикс для image tag переменной |

#### Функциональность

| Параметр | Тип | Default | Описание |
|----------|-----|---------|---------|
| `helm_sync_enabled` | boolean | `false` | Включить helm:sync - синк чарта из внешнего регистри |
| `helm_diff_enabled` | boolean | `true` | Включить helm:diff шаг |
| `helm_secrets_enabled` | boolean | `false` | Использовать Helm secrets плагин |
| `kubedog_enabled` | boolean | `false` | Включить kubedog мониторинг развертывания |
| `helm_rollback_enabled` | boolean | `false` | Atomic rollback при ошибке |
| `helm_sync_repo_url` | string | "" | Путь до регистри oci:// или https:// |

#### Параметры выполнения

| Параметр | Тип | Default | Описание |
|----------|-----|---------|---------|
| `helm_timeout` | string | `15m` | Timeout для Helm операций |
| `helm_options` | string | `` | Дополнительные Helm параметры |
| `helm_image` | string | `${HARBOR_HOST}/...` | Docker image с инструментами |
| `helm_runner_tags` | array | `["infra"]` | GitLab runner tags |
| `debug_mode` | boolean | `false` | Режим отладки (set -x) |

---

## 🎯 Jobs в компоненте

### 1. helm:sync (отдельный job)

**Когда запускается:**
- Если включено `helm_sync_enabled` и указан `helm_sync_repo_url`
- На изменение `.helm/Chart.yaml` при MR


### 2. helm:diff (опциональный)

**Когда запускается:**
- Если `helm_diff_enabled: true` (по умолчанию)
- При MR
- После успешного helm:sync

**Что делает:**
```bash
✅ helm diff upgrade показывает различия
✅ Демонстрирует что изменится
✅ Для review перед deploy
```

**Отключить diff:**
```yaml
include:
  - component: $CI_SERVER_HOST/gitlab-components/helm/gitlab-ci-helm@v1
    inputs:
      helm_diff_enabled: false
```

### 3. helm:deploy (основной)

**Когда запускается:**
- На `CI_COMMIT_TAG`
- Manual trigger (по умолчанию)

**Что делает:**
```bash
✅ helm upgrade -i (deploy или update)
✅ History max: 3 версии
✅ kubedog мониторинг (если enabled)
✅ Atomic rollback (если enabled)

```
---

## 💡 Примеры использования

### Пример 1: Базовое развертывание

```yaml
include:
  - component: $CI_SERVER_HOST/gitlab-components/helm/gitlab-ci-helm@v1
    inputs:
      helm_k8s_name: "prod"
      helm_release_name: "my-app"
      helm_release_namespace: "default"

stages:
  - diff
  - deploy

```

### Пример 2: С куbedog мониторингом

```yaml
include:
  - component: $CI_SERVER_HOST/gitlab-components/helm/gitlab-ci-helm@v1
    inputs:
      helm_k8s_name: "prod"
      helm_release_name: "my-app"
      helm_release_namespace: "default"
      kubedog_enabled: true
      helm_rollback_enabled: true

stages:
  - diff
  - deploy

```

### Пример 3: Мультикластер (staging + prod)

```yaml
include:
  - component: $CI_SERVER_HOST/gitlab-components/helm/gitlab-ci-helm@v1
    inputs:
      helm_release_name: "my-app"
      helm_release_namespace: "default"
      skip_default_jobs: true

stages:
  - diff
  - deploy


# STAGING
diff:staging:
  extends: ".helm:diff"
  variables:
    HELM_K8S_NAME: "staging"

deploy:staging:
  extends: ".helm:deploy"
  variables:
    HELM_K8S_NAME: "staging"

# PRODUCTION
diff:prod:
  extends: ".helm:diff"
  variables:
    HELM_K8S_NAME: "prod"

deploy:prod:
  extends: ".helm:deploy"
  variables:
    HELM_K8S_NAME: "prod"
```


### Пример 4: Без diff (быстрый путь)

```yaml
include:
  - component: $CI_SERVER_HOST/gitlab-components/helm/gitlab-ci-helm@v1
    inputs:
      helm_release_name: "my-app"
      helm_release_namespace: "default"
      helm_k8s_name: "prod"
      helm_diff_enabled: false

stages:
  - deploy

```

### Пример 5: С Helm secrets

```yaml
include:
  - component: $CI_SERVER_HOST/gitlab-components/helm/gitlab-ci-helm@v1
    inputs:
      helm_release_name: "my-app"
      helm_release_namespace: "default"
      helm_k8s_name: "prod"
      helm_secrets_enabled: true
      helm_secrets_file: "secrets.yaml"

```

### Пример 6: Sync внешнего helm chart

```yaml
include:
  - component: $CI_SERVER_HOST/gitlab-components/helm/gitlab-ci-helm@v1
    inputs:
      helm_release_name: "my-app"
      helm_release_namespace: "default"
      helm_k8s_name: "prod"
      helm_sync_enabled: true
      helm_sync_repo_url: "https://helm.goharbor.io"

stages:
  - sync
  - diff
  - deploy

```
при этом фаил `.helm/Chart.yaml` выглядит так

```yaml
apiVersion: v2
appVersion: v1.16.5
dependencies:
  - name: harbor
    version: 1.16.5
    repository: oci://harbor/helm
description: Developer-friendly incident response with brilliant Slack integration
name: oncall
type: application
version: 1.16.5
```
---

## 📋 ПРАВИЛО: КОГДА INPUT, КОГДА VARIABLE

| Сценарий | Используйте | INPUT значения | VARIABLE значения |
|----------|-----------|---------------|--------------------|
| **1 кластер** | INPUT | `helm_k8s_name`, `helm_release_name` | - |
| **2+ кластера** | INPUT + VARIABLE | `helm_release_name` | `HELM_K8S_NAME`, `APP_NAMESPACE` |
| **Разные apps** | INPUT | `helm_release_name`, `helm_dir` | `HELM_K8S_NAME` |
| **Разные версии** | INPUT + VARIABLE | `helm_release_name` | `HELM_VERSION` |

--

## ✅ INPUT ПАРАМЕТРЫ (в include)

Используйте INPUT для значений, которые ОДИНАКОВЫ для всех jobs:

```yaml
include:
  - component: $CI_SERVER_HOST/group/helm-component@v1
    inputs:
      # Общие для всех jobs
      helm_k8s_name: "prod"           # Один кластер
      helm_release_name: "my-app"     # Одно приложение
      helm_dir: ".helm"               # Один путь
      helm_timeout: "15m"             # Один timeout
      helm_diff_enabled: true         # Общая конфиг
      kubedog_enabled: false          # Общая конфиг
      helm_image: "${HARBOR_HOST}/..." # Один image
```
---

## 📌 VARIABLE ПАРАМЕТРЫ (в каждом job)

Используйте VARIABLE для значений, которые РАЗЛИЧАЮТСЯ между jobs:

```yaml
deploy:staging:
  extends: helm:deploy
  stage: deploy
  variables:
    HELM_K8S_NAME: "staging"          # Другой кластер
    APP_NAMESPACE: "staging"          # Другое пространство

deploy:prod:
  extends: helm:deploy
  stage: deploy
  variables:
    HELM_K8S_NAME: "prod"             # Другой кластер
    APP_NAMESPACE: "production"       # Другое пространство
```
---

## 🧪 Troubleshooting

### Ошибка: "HELM_RELEASE_NAME is required"

**Причина:** Параметр не установлен в переменных job'а

**Решение:**

```yaml
include:
  - component: $CI_SERVER_HOST/gitlab-components/helm/gitlab-ci-helm@v1
    inputs:
      helm_release_name: "my-app" # ← Добавить
      helm_release_namespace: "default"
      helm_k8s_name: "prod"
```
или

```yaml
deploy:prod:
  extends: helm:deploy
  variables:
    HELM_RELEASE_NAME: "my-app"  # ← Добавить
    HELM_K8S_NAME: "prod"
```

### Ошибка: "HELM_K8S_NAME is required"

**Причина:** Параметр не установлен

**Решение:**

```yaml
include:
  - component: $CI_SERVER_HOST/gitlab-components/helm/gitlab-ci-helm@v1
    inputs:
      helm_release_name: "my-app"
      helm_release_namespace: "default"
      helm_k8s_name: "prod" # ← Добавить
```
или

```yaml
variables:
  HELM_K8S_NAME: "prod"  # или "staging", "dev"
```

### Ошибка: "KUBE_CONFIG_PROD is not set"

**Причина:** Переменная CI/CD не установлена в GitLab

**Решение:**
1. Идти в Settings → CI/CD → Variables
2. Добавить переменную: `KUBE_CONFIG_PROD`
3. Значение: base64-encoded kubeconfig
4. Mark as protected (если нужно)
5. Mark as masked (если нужно)

### Ошибка: "helm lint failed"

**Причина:** Syntax ошибка в Helm чарте

**Решение:**
```bash
# Локально проверить:
helm lint .helm --strict

# Исправить ошибки в Chart.yaml или templates
```

### Diff не показывает изменения

**Причина:** helm_diff_enabled может быть false

**Решение:**
```yaml
include:
  - component: $CI_SERVER_HOST/gitlab-components/helm/gitlab-ci-helm@v1
    inputs:
      helm_diff_enabled: true  # ← Включить
```
