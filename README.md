# itis-k8s
### Небольщая памятка для воркшопа
#### 0. Подготовка

- скачать ключ для своего пользователя
- сделать правильные права для ключа
    ```
    sudo chmod /папка_где_лежит_ключ_/itis-user000
    ```

- проверить версию питона и установить пакет `pip`
- установить ансибл

    ```
    pip install ansible
    ```
- убедиться в работоспособности ключа
    ```
    ssh itis-user000@ip_адрес -i /папка_где_лежит_ключ_/itis-user000
    ```


#### 1. подготавливаем kubespray
- забираем kubespray из официального репозитория
    ```
    git clone https://github.com/kubernetes-sigs/kubespray.git
    ```
- переключаемся на древнюю, но стабильную ветку
    ```
    git checkout release-2.8
    ```
- заходим в папку с проектом, подготавливаем окружение
    ```
    sudo pip install -r requirements.txt
    ```
- копируем ``inventory/sample`` в ``inventory/mycluster``
    ```
    cp -rfp inventory/sample inventory/mycluster
    ```

- крафтим inventory файл для ансибла, подставляя свои ip-адреса
    ```
    declare -a IPS=(10.10.1.3 10.10.1.4 10.10.1.5)
    ```
    ```
    CONFIG_FILE=inventory/mycluster/hosts.ini python3 contrib/inventory_builder/inventory.py ${IPS[@]}
    ```
- проверяем, что все подставилось верно, добавляем ип адерса

    ```
    nano inventory/mycluster/hosts.ini
    ```

#### 2. подготавливаем плагины
- правим файл `inventory/mycluster/group_vars/all/all.yml`: 
    расскоментируем строку 
    ```
    kube_read_only_port: 10255
    ``` 
- правим файл `inventory/mycluster/group_vars/k8s-cluster/addons.yml`: подключаем helm
    ```  
    helm_enabled: true
    ```
  подключим и расскоментируем блок для метрик-сервера
    ```
    metrics_server_enabled: true
      metrics_server_kubelet_insecure_tls: true
      metrics_server_metric_resolution: 60s
      metrics_server_kubelet_preferred_address_types: "InternalIP"
    ```

  подключим и расскоментируем блок для метрик-сервера
    ```
    ingress_nginx_enabled: true
    ingress_nginx_host_network: true
    ingress_nginx_nodeselector:
      node-role.kubernetes.io/master: ""
    ingress_nginx_tolerations:
      - key: "key"
        operator: "Equal"
        value: "value"
        effect: "NoSchedule"
    ingress_nginx_namespace: "ingress-nginx"
    ingress_nginx_insecure_port: 80
    ingress_nginx_secure_port: 443
    ingress_nginx_configmap:
      map-hash-bucket-size: "128"
      ssl-protocols: "SSLv2"
    ingress_nginx_configmap_tcp_services:
      9000: "default/example-go:8080"
    ingress_nginx_configmap_udp_services:
      53: "kube-system/kube-dns:53"
    ```
#### 3. запускаем плейбук
- запускаем ансибл плейбук

    ```
    ansible-playbook -i inventory/mycluster/hosts.ini -b -u itis-user000 cluster.yml --private-key=/папка_где_лежит_ключ_/itis-user000
    ```
- подключаемся к кластеру, копируем конфиг для `kubectl`
    ```
    ssh itis-user000@ip_адрес -i /папка_где_лежит_ключ_/itis-user000
    mkdir $HOME/.kube/
    sudo cp /etc/kubernetes/admin.conf  $HOME/.kube/config
    sudo chown itis-user000 $HOME/.kube/config
    ```

- проверяем, что кластер работает
    ```
    kubectl get nodes
    ```


#### 4. тестовый веб-сервис

- клонируем к себе на мастер-ноду данный проект
- смотрим, что в файлах `cafe.yaml` и `cafe-ingress.yaml`
- создаем отдельный неймспейс для проекта
    ```
    kubectl create ns test
    ```
- применяем конфигурацию
    ```
    kubectl apply -f cafe.yaml -n test
    kubectl apply -f cafe-ingress.yaml -n test
    ```
- смотрим, что получилось
    ```
    kubectl get pods -n test
    ```
    ```
    kubectl get svc -n test
    ```
    ```
    kubectl get ingress -n test
    ```



- правим у себя на рабочей машине файл `/etc/hosts`,
  он должен содержать днс записи нашего нового тестового хоста и ip-адресов ваших нод

    ```
    172.243.78.168   cafe.mydomain.io
    172.244.27.147   cafe.mydomain.io
    ```
- проверяем браузером работоспособность
#### 5. prometheus и grafana
- скачиваем оператор для prometheus на одну из мастер нод
    ```
    git clone https://github.com/coreos/prometheus-operator.git
    cd prometheus-operator/contrib/kube-prometheus
    ```
- применяем манифесты
    ```
    kubectl create -f manifests/ || true
    until kubectl get customresourcedefinitions servicemonitors.monitoring.coreos.com ; do date; sleep 1; echo ""; done
    until kubectl get servicemonitors --all-namespaces ; do date; sleep 1; echo ""; done
    kubectl create -f manifests/ 2>/dev/null || true
    ```
- смотрим что получилось в неймспейсе monitoring
- добавляем ингресс для графаны из этого проекта
    ```
    kubectl apply -f grafana-ingress.yaml 
    ```
- правим у себя на рабочей машине файл `/etc/hosts`,

    ```
    172.243.78.168   grafana.mydomain.io
    172.244.27.147   grafana.mydomain.io
    ```
- проверяем браузером работу мониторинга


#### 6. loghouse
- ворк ин прогресс

#### примеры взяты из:
- https://github.com/nginxinc/kubernetes-ingress/tree/master/examples/complete-example
- 