# Задание по Argo CD (автодеплой в Kubernetes)

## Цель

Настроить GitOps-деплой так, чтобы изменения в GitHub-репозитории автоматически применялись в кластере.

## Важно: не копировать как есть

Примеры ниже содержат плейсхолдеры. Перед применением обязательно замените:

- `<your-org>`
- `<your-repo>`
- `<your-branch>`
- `<target-namespace>`
- `<argocd-project>`

## Что требуется

1. Установить Argo CD в кластер.
2. Подключить ваш GitHub-репозиторий.
3. Создать `Application` для деплоя `k8s/overlays/dev`.
4. Включить автоматическую синхронизацию (`automated` + `prune` + `selfHeal`).
5. Дополнительно: создать второе приложение для `prod` (по желанию/для бонуса).

## Пример `Application` для `dev`

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: messager-dev
  namespace: argocd
spec:
  project: <argocd-project>
  source:
    repoURL: https://github.com/<your-org>/<your-repo>.git
    targetRevision: <your-branch>
    path: k8s/overlays/dev
  destination:
    server: https://kubernetes.default.svc
    namespace: <target-namespace>
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

## Опциональный `Application` для `prod`

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: messager-prod
  namespace: argocd
spec:
  project: <argocd-project>
  source:
    repoURL: https://github.com/<your-org>/<your-repo>.git
    targetRevision: <your-branch>
    path: k8s/overlays/prod
  destination:
    server: https://kubernetes.default.svc
    namespace: <target-namespace>-prod
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

## Проверки

- статус приложения: `Synced` и `Healthy`
- при изменении в Git (`replicas`, image tag, env) Argo CD применяет изменения автоматически
- при ручном drift в кластере Argo CD возвращает desired state (`selfHeal`)

## Что сдать по этой части

- манифест(ы) `Application`
- скриншот/выгрузку статуса Argo CD
- короткое описание GitOps-процесса для вашего репозитория

## Критерии оценки

- корректная связь Argo CD с Git
- рабочий автосинк без ручного `kubectl apply`
- прозрачный и воспроизводимый процесс деплоя
