# **<p style="text-align: center;">Инструкция по запуску тестового кластера из 2х серверов CentOS с nginx и keepalived</p>**

<image src="./images/top-img.png" alt="top-img">  

### **Задача:** 
Необходимо установить и настроить работу кластера из 2х серверов с nginx для совместной работы с резервированием под управлением keepalived.

### **Для решения задачи будут использоваться:**

- VirtualBox 7.0
- CentOS 7 (CentOS Linux release 7.9.2009)
- Nginx (nginx/1.20.1)
- Keepalived (Keepalived v1.3.5)

### **Схема сети:**

Схема сети в упрощённом виде выглядит следующим образом:

<image src="./images/scheme.jpg" alt="scheme">

**Application** - в роли приложения будет выступать бразузер хоста. На хосте будут запущены витуальные машины с серверами.

**Keepalived** - служба которая реализует протокол VRRP (Virtual Router Redundancy Protocol) для Linux. Демон keepalived будет следить за работоспособностью машин и nginx и в случае обнаружения сбоя — исключает сбойный сервер из списка активных серверов, делегируя его адреса другому серверу.

**CentOS + nginx** - Linux CentOS с веб серверами с разными стартовыми страницами, для проверки работоспособности системы.

### 1. Установка CentOS 7, nginx, keepalived.

#### 1.1 Устанавливаем CentOs 7 на виртуальную машину.

Задаём настройки для виртуальной машины и простой пароль для удобства тестирования:

<image src="./images/centos_install_1.jpg" alt="install">

<image src="./images/centos_install_2.jpg" alt="install">

<image src="./images/centos_install_3.jpg" alt="install">

<image src="./images/centos_install_4.jpg" alt="install">

Запускаем установку:

В ходе установки выбираем опцию Server with GUI для удобства.

<image src="./images/centos_install_5.jpg" alt="install">

Пока идёт установка системы, задаём root пароль и создаём основного пользователя. Пароль пользователя для удобства можно не задавать.

<image src="./images/centos_install_6.jpg" alt="install">

Выполняем обновление системы до актуального состояния командой `yum update`. Если не обновиться, то не установятся корректно гостевые дополнения.

Затем запускаем команду `yum install dkms gcc make kernel-devel bzip2 binutils patch libgomp glibc-headers glibc-devel kernel-headers elfutils-libelf-devel` для корректной установки необходимых утилит для установки гостевых дополнений.

И наконец, запускаем установку гостевых дополнений через меню виртуальной машины:

<image src="./images/centos_install_7.jpg" alt="install">

После установки, диск с гостевыми дополнениями можно извлечь.  
Выключаем виртуальную машину, идём в настройки сети и включаем `Виртуальный адаптер хоста` в качестве типа подключения.

<image src="./images/centos_install_8.jpg" alt="install">

Затем идём настраивать `Виртуальный адаптер хоста`. Его настройки можно найти в меню VirtualBox `Файл` -> `Инструменты` -> `Менеджер сетей`.  
Адрес IPv4 адаптера устанавливается автоматически, но можно изменить чтобы удобно было запоминать. Я установил адрес `192.168.10.1`.

<image src="./images/centos_install_9.jpg" alt="install">

DHCP сервер выключаем.

<image src="./images/centos_install_10.jpg" alt="install">

Запускаем виртальную машину. Теперь, после установки гостевых дополнений, можно нормально ресайзить окно машины и рабочий стол будет сам подстраиваться под размер окна. Так же можно включить двунаправленный буфер обмена и создать общую папку для обмена файлами с хостовой машиной.

<image src="./images/centos_install_11.jpg" alt="install">

#### 1.2 Устанавливаем и настраиваем nginx.

С теми репозиториями, которые по умолчанию `nginx` не установится. Поэтому добавляем новый репозиторий командой `yum install epel-release`.

Затем устанавливаем `nginx` командой `yum install nginx`.

Запускаем `nginx` командой `systemctl start nginx`.

Добавляем `nginx` в автозагрузку командой `systemctl enable nginx`.

Проверяем всё ли работаем командой `systemctl start nginx`.

Проверяем есть ли стартовая страница сервера в браузере по адресу `127.0.0.1`.

<image src="./images/nginx_install_1.jpg" alt="install">

Далее разрешаем http и https трафик командами  
`firewall-cmd --zone=public --permanent --add-service=http`  
и  
`firewall-cmd --zone=public --permanent --add-service=https`. 

Перезагружаем брандмауэр `firewall-cmd --reload`.

#### 1.3 Устанавливаем keepalived.

Устанавливаем сервис `keepalived` командой `yum install keepalived`.

Добавляем следующий параметр Linux Kernel командой `echo "net.ipv4.ip_nonlocal_bind = 1" >> /etc/sysctl.conf`, чтобы включить поддержку плавающих IP.

Добавляем `keepalived` в автозагрузку командой `systemctl enable keepalived`.

Выключаем виртуальную машину.

### 2. Клонирование виртуальных машин, настройка сети и запуск резервирования системы.

Клонируем виртуальную машину. Создаём 2 новые машины с разными названиями. Я сделал `CetnOS_7(10)` и `CentOS_7(20)` с учётом последнего октета ip адресов, которые планирую задавать для организации внутренней сети между машинами. При этом, нужно обязательно указать опцию `Сгенерировать новые MAC-адреса всех сетевых адаптеров`.

<image src="./images/config_clone_1.jpg" alt="clone">

<image src="./images/config_clone_2.jpg" alt="clone">

Запускаем обе машины и идём в настройки сетевых адаптеров.

Необходимо настроить второй адаптер (у меня он называется enp0s8) на обоих машинах.

<image src="./images/config_clone_3.jpg" alt="clone">

Отключаем DHCP и прописываем статические IP адреса для каждой машины.

<image src="./images/config_clone_4.jpg" alt="clone">

<image src="./images/config_clone_5.jpg" alt="clone">

Добавляем маркер в виде IP адреса машины на стартовую страницу nginx, чтобы понимать, какой сервер будет работать через виртуальный IP в данный момент.

Открываем файл `index.html` и добавляем IP куда-нибудь на видное место для обоих виртуальных машин (показано для одной).

`nano /usr/share/nginx/html/index.html`

<image src="./images/config_clone_6.jpg" alt="clone">

<image src="./images/config_clone_7.jpg" alt="clone"> 

Делаем резервную копию файла конфигурации keepalived командой `cp /etc/keepalived/keepalived.conf /etc/keepalived/keepalived.conf.bak` и открываем файл конфигурации для редактирования `nano /etc/keepalived/keepalived.conf`.
Копируем туда конфигурацию для keepalived.
Для MASTER конфигурации (192.168.10.10):

```
global_defs {
  # Keepalived process identifier
  router_id nginx
}

# Script to check whether Nginx is running or not
vrrp_script check_nginx {
  script "pidof nginx"
  interval 2
  weight 50
}

# Virtual interface - The priority specifies the order in which the assigned interface to take over in a failover
vrrp_instance VI_01 {
  state MASTER
  interface enp0s8
  virtual_router_id 151
  priority 110

  # The virtual ip address shared between the two NGINX Web Server which will float
  virtual_ipaddress {
    192.168.10.50/24
  }
  track_script {
    check_nginx
  }
  authentication {
    auth_type AH
    auth_pass secret
  }
}
```

и для BACKUP конфигурации (192.168.10.20):

```
global_defs {
  # Keepalived process identifier
  router_id nginx
}

# Script to check whether Nginx is running or not
vrrp_script check_nginx {
  script "pidof nginx"
  interval 2
  weight 50
}

# Virtual interface - The priority specifies the order in which the assigned interface to take over in a failover
vrrp_instance VI_01 {
  state BACKUP
  interface enp0s8
  virtual_router_id 151
  priority 100

  # The virtual ip address shared between the two NGINX Web Server which will float
  virtual_ipaddress {
    192.168.10.50/24
  }
  track_script {
    check_nginx
  }
  authentication {
    auth_type AH
    auth_pass secret
  }
}
```
Далее разрешаем в брандмауэре протокол VRRP командой  
`firewall-cmd --add-rich-rule='rule protocol value="vrrp" accept' --permanent`,  
 разрешаем передачу трафика для широковещательного адреса 224.0.0.18  
 `firewall-cmd --zone=public --add-rich-rule='rule family="ipv4" destination address="224.0.0.18" protocol value="ip" accept' --permanent`  
 на обоих машинах и перезагружаем брандмауэр `firewall-cmd --reload`.

### 3. Проверка работоспособности:
Проверяем есть ли страница на хостовой машине по адресу `192.168.10.50`. Это наш виртуальный IP. 

<image src="./images/check_1.jpg" alt="check">

Должна появиться стартовая страница nginx с адресом `192.168.10.10` в заголовке т.к. это наш сервер с конфигурацией MASTER.
Выключаем nginx командой `systemctl stop nginx`. Обновляем браузер хоста по адресу `192.168.10.50`.

<image src="./images/check_2.jpg" alt="check">

Должна загрузиться страница с другого сервера (BACKUP). Если снова включить nginx на сервере 192.168.10.10, то keepalived переключит работу на него.  
Таким образом осуществляется резервирование web-серверов по самой простой схеме.