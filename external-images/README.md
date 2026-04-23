# Secure Image Registry

Этот репозиторий служит **единственным источником достоверной информации** (Single Source of Truth) для **внешних контейнеров**, используемых командами в нашей инфраструктуре.

Он обеспечивает безопасную доставку open-source и vendor-образов из публичных реестров (таких как Docker Hub, GHCR, Quay) во внутренний контур компании.

---

## Цель проекта

Использование внешних образов (например, `docker pull nginx`) напрямую в production-среде строго запрещено. Все внешние контейнеры обязаны пройти через данный пайплайн для обеспечения следующих ключевых аспектов:

1.  **Чистота и безопасность**:
    *  Сканирование на наличие уязвимостей (**Trivy**).
    *  Применение дополнительных сканеров безопасности.
    *  Антивирусная проверка.
2.  **Доверие и целостность**:
    *  Каждый образ подлежит обязательной подписи корпоративным ключом (Cosign).
    *  Подписи проходят проверку на этапах сборки, развертывания (деплоя) (внутри Kubernetes (через Admission Controller)).
3.  **Управляемость и контроль**:
    *  Команды должны использовать исключительно зафиксированные (immutable) версии образов.
    *  Использование тега `latest` строго запрещено.
    *  Образы, срок жизни которых превышает 1 год, не допускаются к использованию.

---

## 📂 Структура проекта

Иерархия папок жестко привязана к организационной структуре. Это позволяет изолировать команды и автоматически формировать понятные пути в локальном реестре.

**Структура репозитория:**
```
departments/
└── {подразделение}/
    └── {отдел}/
        └── {команда}/
            └── images.yaml

```

благодорая чему автоматически формируется путь:

- При `INTERNAL_REGISTRY_PROJECT_MODE: hierarchical`: `${HARBOR_HOST}/{подразделение}/{отдел}/{команда}/{image}:{тег}`
- При `INTERNAL_REGISTRY_PROJECT_MODE: ext-images`: `${HARBOR_HOST}/ext-images/{подразделение}/{отдел}/{команда}/{image}:{тег}`


### Пример

```
#INTERNAL_REGISTRY_PROJECT_MODE: hierarchical
${HARBOR_HOST}/infra/postgres:18

#INTERNAL_REGISTRY_PROJECT_MODE: ext-images
${HARBOR_HOST}/ext-images/infra/postgres:18

```

## Формат images.yaml

Только внешние образы из публичных реестров.
Команды управляют своими образами через файл `images.yaml`.

### Правила заполнения
1.  **Только source**: Мы указываем полный путь к внешнему образу.
2.  **Один контейнер — одна запись**: Нельзя добавить две записи с одним и тем же именем контейнера (например, два разных `postgres` в одном файле).
3.  **Нет `latest`**: Использование тега `latest` или отсутствие тега запрещено.
4.  **Актуальность**: Образ не должен быть опубликован более 1 года назад.


✅ Правильные примеры

```
images:
  - description: "PostgreSQL 18.1 для основного кластера"
    source: "docker.io/library/postgres:18.1-alpine"

  # Redis
  - source: "docker.io/library/redis:8.2.1-alpine"

  # Nginx
  - description: "Nginx 1.28.1 для статического хостинга"
    source: "nginx:1.28.1-alpine"
```
❌ Запрещено

```
images:
  # ❌ latest запрещён
  - source: "docker.io/library/nginx:latest"

  # ❌ дубли в ОДНОМ файле (database/images.yaml)
  - source: "docker.io/library/postgres:15.4"
  - source: "ghcr.io/bitnami/postgres:15.4"     # ← ОШИБКА: postgres дублируется

  # ❌ образы старше 1 года (pipeline проверит)
  - source: "docker.io/library/nginx:1.18.0"    # если >1 года → fail

```

# Workflow

## Добавить новый образ

1.  Создайте ветку от `main`.
2.  Откройте `images.yaml` своей команды.
3.  Добавьте новую строку с `source`.
4.  Создайте Merge Request (MR).

## Обновить версию

1.  В `images.yaml` найдите нужный образ.
2.  Измените тег в поле `source` на более новый (например, с `1.0.0` на `1.0.1`).
3.  **Важно:** Система проверит SemVer. Понижение версии (Downgrade) запрещено.


## Удаление образа

1.  Просто удалите строку из `images.yaml`.
    *(Примечание: физическое удаление из Harbor пока не обрабатывается атоматикой)*.


# Описание Pipeline

После создания MR запускается автоматическая проверка:

1.  **DETECT**:
    *   Определяет список измененных образов (Delta).
2.  **VALIDATE**:
    *   Проверка уникальности имен контейнеров в файле.
    *   Проверка отсутствия тегов `latest`.
    *   Проверка SemVer (запрет отката версии).
3.  **APPROVAL**:
    *   Требуется подтверждение от Tech Lead или DevOps.
4.  **SCAN**:
    *   **Trivy|Grype**: Поиск уязвимостей (CRITICAL блокирует пайплайн).
    *   **Anti-Virus**: Проверка файловой системы контейнера.
    *   **Security Scanners**: Дополнительные проверки (misconfiguration).
5.  **PUSH**:
    *   Загрузка в Harbor по целевому пути.
6.  **SIGN**:
    *   Подпись образа ключом компании (Cosign).


# Настройка переменных CI/CD

Для корректной работы маппинга внешних реестров в проекты Harbor Proxy, задайте переменные в GitLab CI/CD Settings:

| Переменная | Пример значения | Описание |
|------------|-----------------|---------|
| PROXY_REGISTRY_HOST | proxy.harbor.company.com | Адрес прокси registry |
| INTERNAL_REGISTRY_HOST | `${HARBROR_HOST}` | Адрес внутреннего registry |
| PROXY_REGISTRY_USER | `${PROXY_REGISTRY_USER}` | Пользователь для прокси |
| PROXY_REGISTRY_PASS | `${PROXY_REGISTRY_PASSWORD}` | Пароль для прокси |
| INTERNAL_REGISTRY_USER |  `${HARBOR_USER}`| Пользователь для internal |
| INTERNAL_REGISTRY_PASS | `${HARBOR_PASSWOR}` | Пароль для internal  |
| REGISTRY_MAPPING_DOCKER_IO | docker-hub | Проект для docker.io |
| REGISTRY_MAPPING_GCR_IO | gcr-proxy | Проект для gcr.io |
| REGISTRY_MAPPING_GHCR_IO | ghcr-proxy | Проект для ghcr.io |
| REGISTRY_MAPPING_QUAY_IO | quay-proxy | Проект для quay.io |
| REGISTRY_MAPPING_REGISTRY_K8S_IO | k8s-proxy | Проект для registry.k8s.io |
| INTERNAL_REGISTRY_PROJECT_MODE | `hierarchical` | **(Default)** Имя проекта = название подразделения (первая папка) |
| | `ext-images` | Все образы в проекте `ext-images`, структура папок сохраняется внутри |
| COSIGN_KEY | base64(...) либо имя в vault | Cosign private key |
| COSIGN_VAULT_URL |  `${VAULT_ADDR}` | адресс vault - https://vault.local |
| COSIGN_VAULT_PATH | "cosign" | путь в vault до проекта transit |


# Возможные ошибки (Troubleshooting)

| Ошибка                                   | Причина                                                     | Решение                                                                                   |
| ---------------------------------------- | ------------------------------------------------------------| ----------------------------------------------------------------------------------------- |
| `ERROR: '...:latest' uses forbidden tag` | Вы указали тег `latest` или забыли тег.                     | Укажите конкретную версию (например, `v1.2.3`).                                           |
| `ERROR: Duplicate container name found`  | В вашем файле два раза встречается один и тот же контейнер. | Оставьте только одну запись. Обновляйте версию в ней.                                     |
| `ERROR: trying to downgrade from X to Y` | Вы пытаетесь понизить версию образа.                        | Проверьте теги. Downgrade запрещен.                                                       |
| `ERROR: Image is older than 1 year`      | Дата создания образа > 365 дней.                            | Найдите более свежий тег или образ. Старые образы небезопасны.                            |
| `ERROR: Image pull failed`               | Образ не существует.                                        | Попробуйте проверить локально `docker pull`                                               |
| `ERROR: No mapping for domain`           | Отсутсвтует папинг для нового реджестри.                    | 1. Добавь проект в proxy-registry 2. Добавь env `REGISTRY_MAPPING_`                       |
| `Trivy found CRITICAL vulnerabilities`   | В образе найдены критические уязвимости.                    | Найдите более свежую версию контенера.                                                    |
