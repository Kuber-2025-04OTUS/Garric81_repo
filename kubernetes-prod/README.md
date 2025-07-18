1. Подготовка инфраструктры.Установка   требуемого  количетва ВМ (Ubuntu 20.04-22.04)
2. Установка и настройка DNS-сервера. Установка и настройка BIND9
Требования:
Наличие сетевого интерфейса c прямым доступом в интернет.
Примечание: Все дальнейшие действия необходимо производить с учетом того, что у нас выкуплено доменное имя *.*-it.ru (для примера) и DomainSSL соответственно сертификат для *.sc.rtk-it.ru.
Инсталляция DNS сервера и настройка BIND9
Авторизуемся используя  любой удобным клиент в целевой сервер и выполняем следующие действия:
•	Установка службы BIND9

Установка и настройка BIND9
Если вход в систему изначально был не с правами root, то запускаем сессию в привилегированном режиме:
sudo -i
Обновляем список доступных пакетов программного обеспечения из официальных репозиториев:
apt update
Устанавливаем службу bind9:
apt install bind9
Разрешаем службу во встроенном firewall:
ufw allow Bind9
Далее, необходимо открыть файл /etc/bind/named.conf.options и внести в него следующие изменения:
*******************************************
options {
        directory "/var/cache/bind";
// Запрашивать у следующих DNS 7.88.8.8; 77.88.8.1 yandex  имена, если не найдено в собственной базе
        forwarders { 7.88.8.8; 77.88.8.1; }; 
// Принимать во внимание любые запросы
        allow-query { any; };
};
*******************************************
Остальное убираем. Сохраняем файл /etc/bind/named.conf.options. Проверяем корректность конфига:
named-checkconf
В результате выполнения команды не должно выводиться сообщений об ошибках:

 $named-checkconf                      #команда проверки корректности внесенных изменений
$systemctl restart bind9               #Перезапустим службу BIND9
 $nslookup ubuntu.com 10.255.30.70               #Для проверки - должен выдать несколько белых IP адресов


Создание своей DNS зоны для сети 10.255.30.x/24

 $nano /etc/bind/named.conf.local
редактируем:
zone "****it.ru" {
type master;
file "/etc/bind/*****.ru";
allow-transfer { 10.255.30.70; };
   also-notify { 10.255.30.70; };
};
Остальное убираем. сохраняем. Выходим.

 $systemctl reload bind9                #Перезапустим службу BIND9

2.1. Создаем свой файл с описанием нашей sc.rtk-it.ru зоны DNS, используя как основу уже имеющийся файл.

 $cp /etc/bind/db.local /etc/bind/****it.ru                             #Копируем с новым именем тот, что уже был в ОС Ubuntu.
Редактируем:
 $nano /etc/bind/****it.ru
*************************************************
;BIND data file for local loopback interface
;
$TTL    604800
;Строки начинающиеся на @ описывают настройки самого DNS сервера (в нашем примере 10.255.30.70)
@       IN      SOA     ******it.ru. root.********it.ru. (
                              3         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
;
@       IN      NS      *****it.ru.
@       IN      A       127.0.1.1
@       IN      AAAA    ::1
;Строки ниже описывают настройки DNS сервера по разрешению имен внутри локальной сети
gitlab.*****it.ru.      IN      A       10.255.30.70
************************************

 

Остальное убираем. Сохраняем.
 $rndc reload                   # перезапустим сервис rndc
 $nslookup gitlab.********it.ru 10.255.30.70     #Проверим работу DNS

Примечание: При добавлении новых машин в сеть для корректного разрешения их имен необходимо по аналогии в файл:  $nano /etc/bind/******it.ru
добавлять необходимые имена (По аналогии последней нижней записи) добавляемых машин и перезапускать сервис:
$rndc reload

3. Установка K8S и Helm для 5-и узлов
Для установки, потребуется 5 отдельных виртуальных машин с ОС 20.04 - 22ю04 LTS . Из них, 2 мастер-ноды (название машин оканчивается на m-01 и m-02) и 3 рабочих ноды (названия машин оканчиваются на w-01, w-02 и w-03). Характеристики каждой машины должны быть не ниже следующих:
Параметр	Минимальное значение	Рекомендуемое значение
Количество ядер, шт.	2	4
Объем ОЗУ, Гб	4	8
Объем диска, Гб	50	50
Тип диска	с производительностью не ниже SATA	SSD
Сеть, МБит/с	1000	1000

Для установки должна использоваться учетная запись с правами root или иная учетная запись с использованием механизма sudo.
В инструкции показан пример установки при наличии доступа в Интернет. В случае его отсутствия, должен быть организован доступ к локальным репозиториям ОС Ubuntu, а также предварительно скачаны необходимые дистрибутивы.
Все машины кластера должны быть добавлены в локальный DNS.
Настройка использования локального DNS
Все действия этого раздела необходимо выполнить на всех ВМ кластера.
Если вход в систему изначально был не с правами root, то запускаем сессию в привилегированном режиме:
sudo -i
На всех ВМ необходимо настроить использование локального DNS. Для этого, необходимо открыть файл /etc/netplan/00-installer-config.yaml на редактирование и внести в него адрес сервера DNS в блок nameservers:
 
Сохраняем файл и обновляем настройки сети:
netplan apply
Далее, необходимо убедиться в том, что локальный DNS используется:
ping gitlab.******it.ru
В результате, сервер Gitlab должен успешно пинговаться по имени:
 
Установка Docker и необходимых утилит
Все действия этого раздела необходимо выполнить на всех ВМ кластера.
Устанавливаем вспомогательные пакеты:
sudo -i
apt-get update && apt-get install ca-certificates && apt-get install curl -y  && apt-get install gnupg  && apt-get install lsb-release
mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | gpg --dearmor -o /etc/apt/keyrings/docker.gpg
echo \
"deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
$(lsb_release -cs) stable" | tee /etc/apt/sources.list.d/docker.list > /dev/null
apt-get update
Делее необходимо посмотреть, какие версии докера доступны в локальном репозитории и выбрать 20.10.7:
apt-cache madison docker-ce
Результат вывода команды:
docker-ce | 5:20.10.8~3-0~ubuntu-focal | https://download.docker.com/linux/ubuntu focal/stable amd64 Packages
docker-ce | 5:20.10.7~3-0~ubuntu-focal | https://download.docker.com/linux/ubuntu focal/stable amd64 Packages
docker-ce | 5:20.10.6~3-0~ubuntu-focal | https://download.docker.com/linux/ubuntu focal/stable amd64 Packages
Устанавливаем версию 20.10.7:
apt-get install docker-ce=5:20.10.7~3-0~ubuntu-focal docker-ce-cli=5:20.10.7~3-0~ubuntu-focal containerd.io docker-compose-plugin -y
systemctl enable docker
systemctl daemon-reload && systemctl restart docker && chmod 777 /var/run/docker.sock && systemctl status docker
Вывод должен быть следующий:
 
Установка K8s (В моем примере ставиться  старая версия  K8S)
Все действия этого раздела необходимо выполнить на всех ВМ.
Будем устанавливать пакет k8s v1.21.3 - необходимо именно версию v1.21.3, версии выше не подойдут.
apt-get update && apt-get upgrade -y
apt-get install curl apt-transport-https git iptables-persistent -y
Важно: в процессе установки iptables-persistent может запросить подтверждение сохранить правила брандмауэра — необходимо выбрать no (отказываемся).
Отключаем файл подкачки:
sudo swapoff -a
Далее, необходимо убрать настройку, чтобы файл подкачки не включался после перезагрузки ОС. Для этого, необходимо открыть файл /etc/fstab на редактирование и закомментировать строку:
/swap.img      none    swap    sw      0       0
Сохраняем файл и выходим.
Устанавливаем вспомогательные пакеты:
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add - && \
echo "deb http://apt.kubernetes.io/ kubernetes-xenial main" | 
tee /etc/apt/sources.list.d/kubernetes.list
apt-get update -q
Необходимо проверить, какие версии доступны с помощью команды ниже:
apt list -a kubeadm
Устанавливаем нужную версию 1.21.3:
apt-get install -qy kubelet=1.21.14-00 kubectl=1.21.14-00 kubeadm=1.21.14-00 docker
Закрепим установленные версии:
apt-mark hold kubeadm kubelet kubectl
Важно: на всех узлах кластера необходимо проверить статус файрвола:
ufw status
Если значение status = inactive, то на всех узлах кластера необходимо разрешить подключения по ssh и включить файрвол:
ufw allow ssh && ufw enable
Далее, на всех узлах кластера разрешаем необходимые порты и протоколы:
ufw allow 6443 & ufw allow 6443/tcp
ufw allow 2379 & ufw allow 2379/tcp
ufw allow 2380  &  ufw allow 2380/tcp
ufw allow 179 &  ufw allow 179/tcp
Проверяем статус, должно быть следующее:
 
Создание кластера
Выполняем команду на мастер ноде с именем, заканчивающимся на m-01 (в примере ниже используется имя devops-test-k8s-m-01.sc.rtk-it.ru):
kubeadm init --control-plane-endpoint "devops-test-k8s-m-01.sc.rtk-it.ru:6443" --upload-certs --kubernetes-version v1.21.14 --v=5
Примечание: в команде выше имя ноды должно быть прописано в записи A используемого DNS сервера и с любой из машин кластера должен идти ping к devops-test-k8s-m-01.sc.rtk-it.ru.
Результат команды (приведен частичный вывод), следует сохранить, для добавления остальных узлов в кластер:
kubeadm join k8s-m-01.******it.ru:6443 --token e8zyca.o7me3qkkmq0beni9 \
        --discovery-token-ca-cert-hash sha256:84696e0701b13d39f0d2c9c7f623008fa766cca27eff1de9846db7c692188f2c \
        --control-plane --certificate-key ff186cdcba0ad177e0c9cbc8455b09c73faae215204185d74b0e4d5b0dede9ad
Please note that the certificate-key gives access to cluster sensitive data, keep it secret!
As a safeguard, uploaded-certs will be deleted in two hours; If necessary, you can use
"kubeadm init phase upload-certs --upload-certs" to reload certs afterward.

Then you can join any number of worker nodes by running the following on each as root:
kubeadm join k8s-m-01.******it.ru:6443 --token e8zyca.o7me3qkkmq0beni9 \
        --discovery-token-ca-cert-hash sha256:84696e0701b13d39f0d2c9c7f623008fa766cca27eff1de9846db7c692188f2c
Необходимо проверить, что файл конфигурации /etc/kubernetes/admin.conf создался.
Далее, необходимо выполнить команды:
mkdir -p $HOME/.kube 
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config 
sudo chown $(id -u):$(id -g) $HOME/.kube/config
Следует сразу разрешить запуск подов на master-нодах, используя команду:
kubectl taint nodes --all node-role.kubernetes.io/master-
Далее, необходимо проверить, что что мастер-нода поднялась:
kubectl get nodes
Вывод должен быть таким:
 
Это нормально, все сделано верно. Статус «NotReady» отображается потому что CNI еще не установлено.
Далее необходимо установить CNI (Container Networking Interface):
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/calico.yaml
После этого необходимо подождать “пару минут” и проверить корректность запуска необходимых подов:
kubectl get pods -A
Вывод должен быть таким:
 
Затем, необходимо присоединить 2-ю мастер-ноду к кластеру. Для этого, на ВМ с именем, оканчивающимся на m-02, необходимо выполнить ранее сохраненную команду:
kubeadm join k8s-m-01.******it.ru:6443 --token e8zyca.o7me3qkkmq0beni9 \
        --discovery-token-ca-cert-hash sha256:84696e0701b13d39f0d2c9c7f623008fa766cca27eff1de9846db7c692188f2c \
        --control-plane --certificate-key ff186cdcba0ad177e0c9cbc8455b09c73faae215204185d74b0e4d5b0dede9ad
Аналогично, на всех рабочих нодах, имена которых оканчиваются на w-01, w-02 и w-03, необходимо выполнить ранее сохраненную команду:
kubeadm join k8s-m-01.********it.ru:6443 --token e8zyca.o7me3qkkmq0beni9 \
        --discovery-token-ca-cert-hash sha256:84696e0701b13d39f0d2c9c7f623008fa766cca27eff1de9846db7c692188f2c
Через несколько минут, на мастер-ноде необходимо проверить, что все ноды корректно подключились:
kubectl get nodes
Вывод должен соответствовать тому, что показан на рисунке ниже:
 
Установка ingress-контроллера в кластер
Исходный файл ingress_controller.yaml.
Действия данного раздела выполняются на мастер-ноде с именем, заканчивающимся на m-01 (в примере ниже используется имя k8s-m-01):
kubectl label node k8s-m-01 run-nginx-ingress=true 
kubectl label node k8s-m-02 run-nginx-ingress=true
Важно: эти команды необходимо выполнять обязательно до применения файла ingress_controller.yaml.
Проверяем наличие меток:
kubectl get nodes --show-labels
Удаление кластера
Для удаления кластера, необходимо выполнить следующие команды:
kubeadm reset
sudo apt-get purge kubeadm kubectl kubelet kubernetes-cni kube*
sudo apt-get autoremove 
sudo rm -rf ~/.kube
Затем, необходимо перезагрузить ОС. 


Обновление нод. Ручное  по  релизное

Снимаем  установленные версии:
apt-mark hold kubeadm kubelet kubectl
Устанавливаем нужную версию 1.22:
apt-get install -qy kubelet=1.22 kubectl=1.22 kubeadm=1.22 docker

Обновлене  нод не ручное  нужно использовать  Kubesparay  * ansible
