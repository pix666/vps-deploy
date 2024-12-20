# Deploy on Debian VPS

Minimum RAM Requirement: 2GB

> **Note**: Before proceeding with this instruction, it may be helpful to replace 
> all occurrences of "example.com" with your actual domain.

 1. Update System
    ```bash
    apt-get update
    apt-get upgrade
    ```

 2. Install Required Packages
    ```bash
    apt-get install -y git make gcc g++ postgresql postgresql-client libpq-dev \
      libjpeg-dev libpng-dev libwebp-dev libgif-dev libtiff-dev zlib1g-dev \
      libxml2-dev libxslt1-dev redis-server redis-tools nginx curl
    ```

 3. Install Python 3.7
    ```bash
    apt-get install -y build-essential gdb lcov pkg-config libbz2-dev libffi-dev libgdbm-dev \
     libgdbm-compat-dev liblzma-dev libncurses5-dev libreadline6-dev libsqlite3-dev libssl-dev \
     lzma lzma-dev tk-dev uuid-dev zlib1g-dev
    ```

    ```bash
    mkdir -p ~/.local/opt

    export VERSION=3.7.17
    wget -qO "/tmp/python-$VERSION.tar.gz" "https://www.python.org/ftp/python/$VERSION/Python-$VERSION.tgz"
    tar -xzf "/tmp/python-$VERSION.tar.gz" -C ~/.local/opt/
    cd ~/.local/opt/Python-$VERSION/
    ./configure --with-pydebug
    make -j $(nproc)
    make altinstall
    ```

 4. Install SASS
    ```bash
    cd ~/.local/opt/
    git clone https://github.com/sass/libsass
    cd ./libsass
    git checkout tags/3.3.3
    make && make install

    cd ..
    git clone https://github.com/sass/sassc
    cd ./sassc
    git checkout tags/3.3.0
    export SASS_LIBSASS_PATH=~/.local/opt/libsass
    make
    cp ./bin/sassc /usr/bin/
    ```

 5. Create User for Web Application
    ```bash
    adduser webapp
    adduser webapp sudo
    ```

 6. Create Project Directory
    ```bash
    mkdir /var/www/
    chown webapp:webapp /var/www/
    ```

 7. Upload Archive to the Server (via FTP or SCP, for example):
    ```bash
    # example:
    scp ./example.com.tar root@127.0.0.1:/var/www/
    ```

 8. Extract Files into Project Directory
    ```bash
    su - webapp
    cd /var/www/

    mkdir example.com
    tar -xf ./example.com.tar -C ./example.com
    cd example.com
    ```

 9. Create PostgreSQL user
    ```bash
    sudo -u postgres createuser --createdb --pwprompt webapp
    ```

10. Restore Database
    ```bash
    createdb example.com
    gunzip -c ./backup/example.com.2023-07-31.sql.gz \
      | grep -v -E '(DROP\ EXTENSION|CREATE\ EXTENSION|DROP\ SCHEMA|CREATE\ SCHEMA|COMMENT\ ON)' \
      | psql -v ON_ERROR_STOP=on --single-transaction --dbname=example.com
    ```

11. Create Virtual Environment
    ```bash
    python3.7 -m venv --prompt=env .venv
    . .venv/bin/activate
    pip install -r requirements/dev.txt
    ```

12. Configure `EMAIL_*` settings in the `src/settings/common.py`.

13. Ensure that the `PROJECT_ROOT` variable in the `src/settings/production.py` file 
corresponds to the absolute path of the project directory.

14. Install uWSGI.
    ```bash
    pip install uwsgi
    ```

    Create a basic uWSGI configuration file at `/var/www/uwsgi.ini`:
    ```uwsgi
    [uwsgi]
    http-socket = 0.0.0.0:%(app_port)
    module = project.wsgi
    master = true
    autoload = true
    protocol = uwsgi
    procname-master = %(app_name)-master
    procname = %(app_name)
    pidfile = /run/uwsgi/%(app_name).pid
    enable-threads = true
    so-keepalive = true
    tcp-nodelay = true
    processes = 2
    threads = 2

    listen = 512
    http-timeout = 300
    socket-timeout = 300
    limit-as = 768
    post-buffering = 204800
    buffer-size = 16384
    max-requests = 5000
    max-worker-lifetime = 3600
    harakiri = 30
    harakiri-verbose = true

    single-interpreter = true
    die-on-term = true
    vacuum = true
    no-orphans = true
    thunder-lock = true

    log-date = true
    log-slow = 3000
    log-zero = true
    log-4xx = true
    log-5xx = true
    log-ioerror = true
    log-sendfile = true
    ```

15. Compile static files.
    ```bash
    export DB_HOST=localhost DB_NAME=example.com DB_USER=webapp DB_PASS=password
    python3 src/manage.py collectstatic --noinput
    python3 src/manage.py compilejsi18n  # (if django_statici18n installed)
    ```
    > Note: Replace the `password` placeholder with the actual password for the `webapp` user in `DB_PASS`.

16. Create a systemd unit file for uWSGI.
    ```bash
    sudo systemctl edit --full --force uwsgi-example.com.service
    ```

    ```systemd
    [Unit]
    Description=uWSGI for example.com

    [Service]
    User=webapp
    WorkingDirectory=/var/www/example.com
    Environment=DB_HOST=localhost DB_NAME=example.com DB_USER=webapp DB_PASS=password
    ExecStart=/var/www/example.com/.venv/bin/uwsgi --ini /var/www/uwsgi.ini --chdir "/var/www/example.com/src" --set-placeholder app_name="example.com" --set-placeholder app_port=8001
    RuntimeDirectory=uwsgi
    Restart=always
    Type=notify
    NotifyAccess=all

    [Install]
    WantedBy=multi-user.target
    ```
    > Note: Replace the `password` placeholder with the actual password for the `webapp` user in `DB_PASS`.

17. Start uWSGI Service.
    ```bash
    sudo systemctl daemon-reload
    sudo systemctl enable uwsgi-example.com.service
    sudo systemctl start uwsgi-example.com.service
    ```

    Check uWSGI Service Status:
    ```bash
    sudo systemctl status uwsgi-example.com.service
    ```

    Test the uWSGI Service Response:
    ```bash
    curl 'http://127.0.0.1:8001/' -H 'Host: example.com'
    ```

    **Note**: If you have Django 1.11 or higher, and you are using HTTPS, you may encounter the error that occurs when submitting forms:
    ```
    Referer checking failed - https://example.com/ does not match any trusted origins.
    ```

    To resolve this issue, add the following line to the `src/settings/production.py` file:
    ```python
    CSRF_TRUSTED_ORIGINS = ["example.com"]
    ```

18. Create Nginx Configuration File.
    ```bash
    sudo cp ./nginx.example.conf /etc/nginx/sites-available/example.com.conf
    sudo ln -s /etc/nginx/sites-available/example.com.conf /etc/nginx/sites-enabled/example.com.conf
    ```

    Check nginx configuration:
    ```bash
    sudo nginx -t
    ```

19. Create a Default Nginx Configuration

    To enhance security, you can create an Nginx configuration that blocks unwanted requests and incorrect routes.

    Create the file `/etc/nginx/sites-available/default.conf` with the following content:
    ```nginx
    server {
      listen        80 default_server;
      listen        [::]:80 default_server;
      listen        443 default_server http2;
      listen        [::]:443 default_server http2;
      ssl_reject_handshake on;
      server_name   _;
      return        444;
    }
    ```

    Activate the configuration:
    ```bash
    sudo ln -s /etc/nginx/sites-available/default.conf /etc/nginx/sites-enabled/
    ```

    Check nginx configuration:
    ```bash
    sudo nginx -t
    ```

    Restart nginx:
    ```bash
    sudo service nginx restart
    ```

20. (Optional) Install and Configure `ufw` Firewall
    ```bash
    sudo apt-get install -y ufw
    ```

    Allow SSH connections so you don’t lose access to the server:
    ```bash
    sudo ufw allow ssh
    ```

    Allow access to HTTP and HTTPS ports for web applications:
    ```bash
    sudo ufw allow http
    sudo ufw allow https
    ```

    Enable the firewall:
    ```bash
    sudo ufw enable
    ```
