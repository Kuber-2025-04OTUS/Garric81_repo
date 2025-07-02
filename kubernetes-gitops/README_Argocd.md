Установка ArgoCD to K8S  
kubectl create namespace argocd

Установка ArgoCD через KubeCTL
Первое что необходимо, так — это поставить данную утилиту под названием kubectl!

Выполняем деплой:
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml



Пример

Переадресация портов¶
Переадресацию портов Kubectl также можно использовать для подключения к серверу API без раскрытия сервиса.


kubectl port-forward svc/argocd-server -n argocd 8080:443


Войдите в систему с помощью CLI¶
Первоначальный пароль для adminучетной записи автоматически генерируется и сохраняется в виде открытого текста в поле passwordв секрете, указанном argocd-initial-admin-secret в пространстве имен установки вашего компакт-диска Argo. Вы можете просто получить этот пароль с помощью argocdCLI:


argocd admin initial-password -n argocd

Создаю  новый проект  для  теста в GitHUB https://github.com/Garric81/ArgoCD/blob/main/README.md
обязательно  что бы  ArgoCD мог  подключиться  к проекту  проект не  должен  быть  пустым

Создаю ключи  для  доступа  по ssh  c ArgoCD  до  проекта https://github.com/Garric81/ArgoCD/blob/main/README.md

root@sc-k8s-master1:/home/garric/argocd# ls -lah
total 24K
drwxr-xr-x  2 root   root   4.0K Jul  2 14:20 .
drwxr-x--- 10 garric garric 4.0K Jul  2 12:19 ..
-rw-------  1 root   root   2.6K Jul  2 14:20 argocd

-rw-r--r--  1 root   root    573 Jul  2 14:20 argocd.pub

root@sc-k8s-master1:/home/garric/argocd# mc
MoTTY X11 proxy: No authorisation provided

root@sc-k8s-master1:/home/garric/argocd#

Переношу  PUB ключ в GitHUB

зарытый  ключ переношу в argocd ---setting ---repository ----connect repo

