# flask-db-test
Basic flask web app for testing database configurations.

This is a basic web app for quickly switching between different database configurations to test a flask deployment.  The app exposes a single database model allowing you to test the ability to read from and write to a database table.  For a lengthier explanation, please read [this blog post](http://czep.net/15/adventures-with-flask.html).

# Install

Assuming CentOS 7, Apache, and mod_wsgi.  For other deployment environments read the docs, fancy pants!

    sudo yum -y update
    sudo yum install vim httpd python python-devel python-virtualenv httpd-devel mod_wsgi

    # needed to compile the psycopg2 adapter
    sudo yum groupinstall development

    sudo systemctl start httpd
    sudo systemctl enable httpd

## Apache

Prepare the /var/www directory.

    sudo mkdir /var/www/{wsgi-scripts,apps}

    # setup webserver admin group and directories
    sudo groupadd www
    sudo usermod -a -G www yourusername
    sudo chown -R root:www /var/www
    sudo chmod 2775 /var/www
    find /var/www -type d -exec sudo chmod 2775 {} +
    find /var/www -type f -exec sudo chmod 0664 {} +


Apache configuration.

    sudo vim /etc/httpd/conf/httpd.conf

Embedded mode

To run mod_wsgi in embedded mode, add the following lines to the end of httpd.conf:

    WSGIScriptAlias /flasktest /var/www/wsgi-scripts/flask-db-test.wsgi

    <Directory /var/www/wsgi-scripts>
        Require all granted
    </Directory>

Daemon mode (recommended)

To run mod_wsgi in daemon mode, also include these lines:

    WSGIDaemonProcess dev \
        processes=2 \
        threads=15 \
        stack-size=524288 \
        maximum-requests=1000 \
        display-name=%{GROUP}
    WSGIProcessGroup dev


## PostgreSQL

    sudo vim /etc/yum.repos.d/CentOS-Base.repo
    # add to [base] and [updates] sections:
    exclude=postgresql*

    sudo yum localinstall http://yum.postgresql.org/9.4/redhat/rhel-7-x86_64/pgdg-centos94-9.4-1.noarch.rpm
    sudo yum install postgresql94.x86_64 \
                     postgresql94-contrib.x86_64 \
                     postgresql94-libs.x86_64 \
                     postgresql94-odbc.x86_64 \
                     postgresql94-server.x86_64 \
                     postgresql94-test.x86_64 \
                     postgresql94-debuginfo.x86_64

    sudo /usr/pgsql-9.4/bin/postgresql94-setup initdb
    sudo systemctl enable postgresql-9.4.service
    sudo systemctl start postgresql-9.4

    sudo vim /var/lib/pgsql/9.4/data/pg_hba.conf
    local   flasktestdb     flasktest   md5

    sudo systemctl restart postgresql-9.4
    sudo /bin/bash
    sudo -u postgres psql

    create role flasktest with login encrypted password 'test12345';
    create database flasktestdb with owner flasktest;

## SQLite

No special installation necessary as it is packaged along with flask.

    sudo chgrp apache app-dev.db 
    chmod 774 app-dev.db
    sudo setenforce 0
    sudo chgrp apache .

Warning: don't leave SELinux in permissive mode!


## Flask

    # install git
    sudo yum install git

    # create a symlink to pg_config
    sudo ln -s /usr/pgsql-9.4/bin/pg_config /usr/local/bin/pg_config

    # set selinux contexts for apache to use psycopg2
    sudo semanage fcontext -a -t httpd_sys_script_exec_t /var/www/apps/flask-db-test/venv/lib/python2.7/site-packages/psycopg2/_psycopg.so
    sudo restorecon -v /var/www/apps/flask-db-test/venv/lib/python2.7/site-packages/psycopg2/_psycopg.so

Install flask-db-test

    cd /var/www/apps
    mkdir flask-db-test
    cd flask-db-test
    git clone ...
    mv flask-db-test.wsgi ../../wsgi-scripts
    virtualenv venv
    source venv/bin/activate
    pip install -r requirements.txt

# Usage

Edit or add new configuration files in local/.  To activate a config, create a symlink from config_live.py pointing to the desired config file.

    ln -s local/config_postgres.py config_live.py

Initialize the database.

    source venv/bin/activate
    python manage.py init_database

If all goes well, touch the wsgi script or reload apache which will instruct modwsgi to reload the application.

    # mod_wsgi in embedded mode
    sudo systemctl restart httpd

    # mod_wsgi in daemon mode
    touch ../../wsgi-scripts/flask-db-test.wsgi

