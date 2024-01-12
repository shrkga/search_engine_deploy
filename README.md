# Search Engine Deploy

Репозиторий содержит код и CI/CD pipeline для развертывание приложения в виде Helm Chart'а. Доставка осуществляется в кластер Kubernetes.

Код микросервисов `crawler` и `ui` вместе с пайплайнами сборки находятся в собственных GitLab репозиториях `search_engine_crawler` и `search_engine_ui` соответственно.

![Список проектов](img/repos.png)

Репозитории `cert-manager` и `monitoring` содержат k8s манифесты, пайплайны и прочие файлы для развертывания в Kubernetes кластере приложения cert-manager и стека приложений для мониторинга (Grafana, Loki, Prometheus) соответственно.

## Хостинг репозиториев Git, CI/CD, registry

Для хранения и сборки проектов, а также хранения артефактов, развернут GitLab через официальный Helm Chart.

```
helm repo add gitlab https://charts.gitlab.io/
helm repo update

helm upgrade --install gitlab gitlab/gitlab \
  --namespace gitlab --create-namespace \
  --timeout 600s \
  --set global.hosts.domain=otus.kga.spb.ru \
  --set global.hosts.https=true \
  --set global.ingress.configureCertmanager=true \
  --set certmanager-issuer.email=shr@kga.spb.ru \
  --set gitlab.gitlab-shell.enabled=false \
  --set global.shell.port=2222 \
  --set global.kas.enabled=true \
  --set global.edition=ce \
  --set global.time_zone=Europe/Moscow \
  --set minio.replicas=1 \
  --set gitlab.gitlab-shell.replicaCount=1 \
  --set gitlab.sidekiq.replicas=1 \
  --set gitlab.webservice.replicaCount=1 \
  --set gitlab.webservice.workerProcesses=1 \
  --set postgresql.image.tag=13.6.0

kubectl get secret -n gitlab gitlab-gitlab-initial-root-password -ojsonpath='{.data.password}' | base64 --decode ; echo
```

> Проекты доступны по адресу https://gitlab.otus.kga.spb.ru/otus

## Хостинг приложения, мониторинг, TLS сертификаты

Для приложения развернут отдельный Kubernetes кластер с целью отработки навыков интеграции GitLab с внешними системами. Кластер с приложением подключен через GitLab Agent. Сам агент установлен через репозиторий проекта `search_engine_deploy`. В настройках агента `search_engine_deploy/.gitlab/agents/yc-k8s/config.yaml` разрешен доступ для всех проектов группы `otus`:
```
ci_access:
  groups:
    - id: otus
```

Переменная `KUBE_CONTEXT` с именем контекста Kubernetes кластера для подключения из GitLab Runner'ов определена для группы проектов `otus`. Переменной присвоен флаг `Protected`, чтобы выкатка в прод осуществлялась только для защищенных тэгов с указанием версии `v*`.

![Переменная KUBE_CONTEXT](img/context.png)

### Ingress-NGINX и Cert-manager

> Репозиторий https://gitlab.otus.kga.spb.ru/otus/cert-manager

Подключение к веб-интерфейсам приложения и системы мониторинга осуществляется через сетевой балансировщик на базе контроллера Ingress-NGINX по протоколу `https` с использованием TLS сертификатов Let's Encrypt.

Для этой цели первым делом в кластере устанавливается Ingress-NGINX контроллер из официального Helm Chart'а:
```
helm upgrade --install ingress-nginx ingress-nginx \
  --repo https://kubernetes.github.io/ingress-nginx \
  --namespace ingress-nginx --create-namespace
```

Следующим шагом осуществляется развертывание в кластер дополнения `Cert-manager` и объявление ресурсов центров сертификации (CA) типа `ClusterIssuer`. В данном проекте используется ClusterIssuer т.к. данный ресурс является глобальным объектом кластера, и область его видимости распространяется на все пространства имен.

Чтобы запустился пайплайн автоматического развертывания в кластере Cert-manager, необходимо выполнить `git push` в репозиторий `cert-manager` с тэгом соответствующим версии релиза данного дополнения. На текущий момент Latest релизом является `v1.13.3` (см. <https://github.com/cert-manager/cert-manager/releases/latest>). В результате выполнятся стадии пайплайна `deploy` (развернуть, или обновить Cert-manager) и `apply` (применить манифесты ресурсов ClusterIssuer). Если запушить в репозиторий изменения без тэга, выполняется только стадия `apply`.

### Search Engine Crawler

> Репозиторий <https://gitlab.otus.kga.spb.ru/otus/search_engine_crawler> <br>
> Образ `registry.otus.kga.spb.ru/otus/search_engine_crawler`

CI/CD пайплайн содержит следующие стадии:
- `unit-test` -- тестирование и генерация отчета о покрытии кода тестами;
- `build` -- сборка образа и размещение в Container Registry с тэгом `${CI_COMMIT_SHA}`;
- `test` -- шаг для тестирования образа, в нашем случае с учетом старого кода и потенциального наличия уязвимостей здесь ничего не делается;
- `release` -- публикация в Container Registry образа `search_engine_crawler` с тэгами `latest` и версией, взятой из файла `VERSION`.

Последний шаг `release` выполняется только при внесении изменений в ветку `main`.

При сборке образа в Dockerfile задаются значения по умолчанию для сканируемого URL'а (передается в качестве аргумента командной строки через `CMD`) и маска для исключения (переменная среды `EXCLUDE_URLS`):
```
FROM alpine:3.9
...
ENV EXCLUDE_URLS=.*github.com
ENTRYPOINT [ "python3", "-u", "crawler/crawler.py" ]
CMD [ "https://vitkhab.github.io/search_engine_test_site/" ]
```

![Пайплайн Crawler](img/crawler-ci.png)

### Search Engine UI

> Репозиторий <https://gitlab.otus.kga.spb.ru/otus/search_engine_ui> <br>
> Образ `registry.otus.kga.spb.ru/otus/search_engine_ui`

Аналогично компоненту Crawler, CI/CD пайплайн содержит следующие стадии:
- `unit-test` -- тестирование и генерация отчета о покрытии кода тестами;
- `build` -- сборка образа и размещение в Container Registry с тэгом `${CI_COMMIT_SHA}`;
- `test` -- шаг для тестирования образа, в нашем случае с учетом старого кода и потенциального наличия уязвимостей здесь ничего не делается;
- `release` -- публикация в Container Registry образа `search_engine_ui` с тэгами `latest` и версией, взятой из файла `VERSION`.

Последний шаг `release` выполняется только при внесении изменений в ветку `main`.

![Пайплайн UI](img/ui-ci.png)

![Telegram Bot](img/tg-bot.jpg)
