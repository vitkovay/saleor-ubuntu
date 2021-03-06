Разворачиваем Saleor на сервере Ubuntu 16.04
https://github.com/mirumee/saleor

Развертывание производил на виртульном сервере reg.ru с память 2048 МБ.
При попытке установки "npm run build-assets" сервер "убивал" процесс установки.
_______________________________________________________________
Установка Node.js:
sudo apt install curl
curl -sL https://deb.nodesource.com/setup_10.x | sudo -E bash -
sudo apt install nodejs

Установка Postgresql:
sudo apt-get update
sudo apt-get install postgresql postgresql-contrib

Устанавливаем дополнительные модули:
sudo apt-get install build-essential python3-dev python3-pip python3-cffi libcairo2 libpango-1.0-0 libpangocairo-1.0-0 libgdk-pixbuf2.0-0 libffi-dev shared-mime-info
sudo apt-get install git

Устанавливаем модуль виртуального окружения:
sudo apt-get update
sudo apt-get install python3-pip
sudo -H pip3 install --upgrade pip
sudo -H pip3 install virtualenv virtualenvwrapper
echo "export VIRTUALENVWRAPPER_PYTHON=/usr/bin/python3" >> ~/.bashrc
echo "export WORKON_HOME=~/Env" >> ~/.bashrc
echo "source /usr/local/bin/virtualenvwrapper.sh" >> ~/.bashrc
source ~/.bashrc

Копируем репозиторий с GitHub:
git clone https://github.com/mirumee/saleor.git

Создаем виртуальное окружение:
mkvirtualenv saleor

Запускаем установку всех необходимых модулей:
cd saleor/
pip install -r requirements.txt

Добавляем настройки, в виртуальное окружение:
export SECRET_KEY=YourSecretKey
Здесь надо указать IP сервера
export ALLOWED_HOSTS=127.0.0.1

Создаем пользователя и базу данных. Пароль надо указать «saleor».
sudo -i -u postgres
createuser --superuser --pwprompt saleor
createdb saleor
exit

Подготавливаем базу данных:
python manage.py migrate

Устанавливаем дополнительные компоненты:
npm install
npm audit fix
npm run build-assets
npm run build-emails

Создаем учетную запись администратора:
python manage.py createsuperuser

Проверяем работоспособность сервера:
python manage.py runserver 0.0.0.0:80

Через браузер открываем сайт по Вашему IP.
Если всё работает, то идем дальше.

Выключаем виртуальное окружение:
deactivate

Устанавливаем дополнительные компоненты:
sudo apt-get install python3-dev
sudo -H pip3 install uwsgi
sudo mkdir -p /etc/uwsgi/sites
sudo apt-get install nano

Создаем файл «saleor.ini» со следующим содержанием:
sudo nano /etc/uwsgi/sites/saleor.ini
————————————
[uwsgi]

chdir = /root/saleor
home = /root/Env/saleor
module = saleor.wsgi:application

master = true
processes = 5

socket = /run/uwsgi/saleor.sock
chown-socket = root:www-data
chmod-socket = 660
vacuum = true
————————————

Создаем файл «uwsgi.service» со следующим содержанием:
sudo nano /etc/systemd/system/uwsgi.service
————————————
[Unit]
Description=uWSGI Emperor service

[Service]
ExecStartPre=/bin/bash -c 'mkdir -p /run/uwsgi; chown root:www-data /run/uwsgi'
ExecStart=/usr/local/bin/uwsgi --emperor /etc/uwsgi/sites
Restart=always
KillSignal=SIGQUIT
Type=notify
NotifyAccess=all
 
Environment="SECRET_KEY=YourSecretKey"
Environment="ALLOWED_HOSTS=127.0.0.1»
Environment="DEFAULT_CURRENCY=RUB"

[Install]
WantedBy=multi-user.target
————————————

Устанавливаем nginx:
sudo apt-get install nginx

Создаем файл «saleor» со следующим содержанием:
sudo nano /etc/nginx/sites-available/saleor
————————————
server {
    listen 80;
    server_name 127.0.0.1;

    #error_log  /root/logs/error.log;

    client_max_body_size 32m;

    location = /favicon.ico { access_log off; log_not_found off; }

    location /static {
        alias /var/www/html/saleor/static/;
    }

    location /media {
        alias /var/www/html/saleor/media/;

    }

    location / {
        include         uwsgi_params;
        uwsgi_pass      unix:/run/uwsgi/saleor.sock;
    }
}
————————————

Необходимо откорректировать settings.py:
MEDIA_ROOT = '/var/www/html/saleor/media/'
STATIC_ROOT = '/var/www/html/saleor/static/'

Надо произвести перенос статических файлов:
workon saleor
~/saleor/python manage.py collectstatic
deactivate


Запускаем сервер:
sudo ln -s /etc/nginx/sites-available/saleor /etc/nginx/sites-enabled
sudo nginx -t
sudo systemctl restart nginx
sudo systemctl start uwsgi
sudo systemctl enable nginx
sudo systemctl enable uwsgi

Проверяем, должно всё работать.
