SELINUX  
Запуск nginx на нестандартном порту 3-мя разными способами 

Задание выполняется на виртуальной машине с установленным nginx, который работает на порту TCP 4881. Порт
TCP 4881 уже проброшен до хоста. SELinux включен.

---------- СПОСОБ 1 ----------
(разрешим работу nginx на порту TCP 4881 c помощью переключателей setsebool)

Все действия выполняются от пользователя root. Переходим в root пользователя:
sudo -i

Установить пакет аудита SELinux core policy utilities:
yum install policycoreutils-python

Проверить, что отключен firewalld:
systemctl status firewalld

Проверить, что режим  SELinux выставлен на Enforcing:
getenforce

Находим в логах информацию о блокировании порта:
less (/var/log/audit/audit.log)
/ 4881

Копируем время, в которое был записан этот лог и,
с помощью утилиты audit2why, смотрим информации о запрете:
grep 1657099333.635:868 /var/log/audit/audit.log | audit2why

Вывод показывает, что нужно поменять параметр nis_enabled:
setsebool -P nis_enabled on

Проверим статус параметр nis_enabled:
getsebool -a | grep nis_enabled

Рестартуем nginx:
systemctl restart nginx

Убедиться, что nginx стал активен:
systemctl status nginx

Вернём запрет работы nginx на порту 4881 обратно:
setsebool -P nis_enabled off

Рестартуем nginx:
systemctl restart nginx


---------- СПОСОБ 2 ----------
(добавим нестандартный порт в имеющийся тип)

Посмотрим имеющиеся типы для http трафика:
semanage port -l | grep http

Добавим порт в тип http_port_t:
semanage port -a -t http_port_t -p tcp 4881

Проверим:
semanage port -l | grep http_port_t

Рестартуем nginx:
systemctl restart nginx

Убедиться, что nginx стал активен:
systemctl status nginx

Удалить нестандартный порт из имеющегося типа:
semanage port -d -t http_port_t -p tcp 4881

Проверим:
semanage port -l | grep http_port_t

Рестартуем nginx:
systemctl restart nginx


---------- СПОСОБ 3 ----------
(Разрешим работу nginx c помощью формирования и установки модуля SELinux:)

Попробуем снова запустить nginx:
systemctl start nginx

SELinux блокирует nginx. Посмотрим логи SELinux, которые относятся к nginx:
grep nginx /var/log/audit/audit.log

Воспользуемся утилитой audit2allow для того, чтобы на основе логов
SELinux сделать модуль, разрешающий работу nginx на нестандартном
порту:
grep nginx /var/log/audit/audit.log | audit2allow -M nginx

Audit2allow сформировал модуль, и сообщил нам команду,
с помощью которой можно применить данный модуль:
semodule -i nginx.pp

Попробуем снова запустить nginx:
systemctl start nginx

Убедиться, что nginx стал активен (изменения сохранятся после перезагрузки):
systemctl status nginx

Просмотреть все установленные модули:
semodule -l

Для удаления модуля воспользуемся командой:
semodule -r nginx
