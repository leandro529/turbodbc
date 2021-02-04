os-base:
    ARG CODE_NAME=focal
    FROM ubuntu:$CODE_NAME

    ENV DEBIAN_FRONTEND=noninteractive
    # set locale to utf-8, which is required for some odbc drivers (mysql);
    # also, set environment as set after 'source /venv/bin/activate'
    ENV LC_ALL=C.UTF-8
    ENV VIRTUAL_ENV=/venv
    ENV PATH=/venv/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin

    RUN apt-get update && apt-get upgrade -y && \
        apt-get install -y build-essential zlib1g-dev libncurses5-dev libgdbm-dev libnss3-dev \
        libssl-dev libreadline-dev libffi-dev libsqlite3-dev \
        wget unixodbc unixodbc-dev libboost-all-dev cmake g++ \
        odbc-postgresql postgresql-client ninja-build gnupg apt-transport-https && \
        apt-get clean

    # hack to cache the docker setup and the image pull
    WITH DOCKER --pull postgres:11 --pull mysql:8 --pull mcr.microsoft.com/mssql/server:2019-latest
        RUN echo ""
    END


driver:
    ARG PYTHON_VERSION=3.8.6
    ARG CODE_NAME=focal
    FROM --build-arg CODE_NAME="$CODE_NAME" +os-base

    # install the mysql odbc driver in /opt
    RUN cd /opt && \
        wget -q https://dev.mysql.com/get/Downloads/Connector-ODBC/8.0/mysql-connector-odbc-8.0.23-linux-glibc2.12-x86-64bit.tar.gz && \
        echo "9b90199bfc8d09a18e1748e979593469 mysql-connector-odbc-8.0.23-linux-glibc2.12-x86-64bit.tar.gz" | md5sum -c - && \
        tar xzf mysql-connector-odbc-8.0.23-linux-glibc2.12-x86-64bit.tar.gz && \
        /opt/mysql-connector-odbc-8.0.23-linux-glibc2.12-x86-64bit/bin/myodbc-installer -d -a -n MySQL -t DRIVER=`ls /opt/mysql-*/lib/libmyodbc*w.so` && \
        rm /opt/*.tar.gz

    # install the mssql odbc driver:
    RUN wget -q https://packages.microsoft.com/keys/microsoft.asc -O- | apt-key add - && \
        wget -q https://packages.microsoft.com/config/debian/9/prod.list -O- > /etc/apt/sources.list.d/mssql-release.list && \
        apt-get update && \
        ACCEPT_EULA=Y apt-get install msodbcsql17 mssql-tools && \
        odbcinst -i -d -f /opt/microsoft/msodbcsql17/etc/odbcinst.ini

    # install the exasol odbc driver:
    RUN cd /opt && \
        wget -q https://www.exasol.com/support/secure/attachment/111075/EXASOL_ODBC-6.2.9.tar.gz && \
        tar xzf EXASOL_ODBC-6.2.9.tar.gz && \
        echo "\n[EXASOL]\nDriver=`ls /opt/EXA*/lib/linux/x86_64/libexaodbc-uo2214lv1.so`\nThreading=2\n" >> /etc/odbcinst.ini && \
        rm /opt/*.tar.gz

python:
    ARG PYTHON_VERSION=3.8.6
    ARG CODE_NAME=focal
    ARG ARROW_VERSION_RULE="<2.0.0"
    ARG NUMPY_VERSION_RULE=""
    ARG PANDAS_VERSION_RULE=""

    FROM --build-arg CODE_NAME="$CODE_NAME" +driver

    RUN apt-get upgrade -y python`echo "$PYTHON_VERSION" | sed 's/\.[^\.]*$//'`-dev
    RUN cd /opt && \
        wget -q https://www.python.org/ftp/python/$PYTHON_VERSION/Python-$PYTHON_VERSION.tgz
    RUN cd /opt && \
        tar xf Python-$PYTHON_VERSION.tgz && \
        cd Python-$PYTHON_VERSION && \
        ./configure -enable-optimizations --enable-shared && \
        make -j4 install
    RUN python3 --version

    RUN cd /opt && \
        wget -q https://github.com/pybind/pybind11/archive/v2.6.2.tar.gz && \
        tar xvf v2.6.2.tar.gz

    # create a python virtualenv at /venv that contains the necessary build and test packages:
    RUN python3 -m venv /venv && \
        /venv/bin/pip install \
            "numpy$NUMPY_VERSION_RULE" \
            "pandas$PANDAS_VERSION_RULE" \
            "pyarrow$ARROW_VERSION_RULE" \
            pybind11==2.6.2 pytest mock pytest-cov

    RUN pyarrow_path=`python -c 'import pyarrow; print(pyarrow.__path__[0])'` && \
        pyarrow_so_version=`python -c 'import pyarrow; print(pyarrow.cpp_build_info.so_version)'` && \
        ln -s $pyarrow_path/libarrow.so.${pyarrow_so_version} $pyarrow_path/libarrow.so && \
        ln -s $pyarrow_path/libarrow_python.so.${pyarrow_so_version} $pyarrow_path/libarrow_python.so

build:
    ARG PYTHON_VERSION=3.8.6
    ARG CODE_NAME=focal
    ARG ARROW_VERSION_RULE="<2.0.0"
    ARG NUMPY_VERSION_RULE=""
    ARG PANDAS_VERSION_RULE=""
    FROM --build-arg CODE_NAME="$CODE_NAME" \
        --build-arg PYTHON_VERSION="$PYTHON_VERSION" \
        --build-arg ARROW_VERSION_RULE="$ARROW_VERSION_RULE" \
        +python

    COPY . /src

    ENV ODBCSYSINI=/src/travis/odbc
    ENV TURBODBC_TEST_CONFIGURATION_FILES="query_fixtures_postgresql.json,query_fixtures_mssql.json,query_fixtures_mysql.json"

    RUN mkdir /src/build
    WORKDIR /src/build
    RUN ln -s /opt/pybind11-2.6.2 /src/pybind11
    RUN cmake -DBUILD_COVERAGE=ON -DCMAKE_INSTALL_PREFIX=./dist -DPYBIND11_PYTHON_VERSION=`echo "$PYTHON_VERSION" | sed 's/\.[^\.]*$//'` -DDISABLE_CXX11_ABI=ON ..
    RUN make -j4
    RUN cmake --build . --target install
    SAVE ARTIFACT /src/build /build

test:
    ARG PYTHON_VERSION=3.8.6
    ARG CODE_NAME=focal
    ARG ARROW_VERSION_RULE="<2.0.0"
    ARG NUMPY_VERSION_RULE=""
    ARG PANDAS_VERSION_RULE=""
    FROM --build-arg CODE_NAME="$CODE_NAME" \
        --build-arg PYTHON_VERSION="$PYTHON_VERSION" \
        --build-arg ARROW_VERSION_RULE="$ARROW_VERSION_RULE" \
        +build

    WITH DOCKER --compose ../earthly-docker-compose.yml
        RUN /bin/bash -c "(r=10;while ! pg_isready --host=localhost --port=5432 --username=postgres ; do ((--r)) || exit; sleep 1 ;done)" && \
            /opt/mssql-tools/bin/sqlcmd -S localhost -U SA -P 'StrongPassword1' -Q 'SELECT @@VERSION' || sleep 10 && \
            /opt/mssql-tools/bin/sqlcmd -S localhost -U SA -P 'StrongPassword1' -Q 'CREATE DATABASE test_db' && \
            ctest --verbose
    END

test-python3.6:
    ARG ARROW_VERSION_RULE="<2.0.0"

    BUILD --build-arg CODE_NAME="bionic" \
        --build-arg PYTHON_VERSION="3.6.12" \
        --build-arg ARROW_VERSION_RULE="$ARROW_VERSION_RULE" \
        --build-arg NUMPY_VERSION_RULE="<1.20.0" \
        --build-arg PANDAS_VERSION_RULE="<1.2.0" \
        +test
    BUILD --build-arg CODE_NAME="xenial" \
        --build-arg PYTHON_VERSION="3.6.12" \
        --build-arg ARROW_VERSION_RULE="$ARROW_VERSION_RULE" \
        --build-arg NUMPY_VERSION_RULE="<1.20.0" \
        --build-arg PANDAS_VERSION_RULE="<1.2.0" \
        +test

test-python3.8:
    ARG ARROW_VERSION_RULE="<2.0.0"

    BUILD --build-arg CODE_NAME="focal" \
        --build-arg PYTHON_VERSION="3.8.5" \
        --build-arg ARROW_VERSION_RULE="$ARROW_VERSION_RULE" \
        --build-arg NUMPY_VERSION_RULE=">=1.20.0" \
        --build-arg PANDAS_VERSION_RULE=">=1.2.1" \
        +test
    BUILD --build-arg CODE_NAME="bionic" \
        --build-arg PYTHON_VERSION="3.8.5" \
        --build-arg ARROW_VERSION_RULE="$ARROW_VERSION_RULE" \
        --build-arg NUMPY_VERSION_RULE=">=1.20.0" \
        --build-arg PANDAS_VERSION_RULE=">=1.2.1" \
        +test

docker:
    ARG PYTHON_VERSION=3.8.6
    ARG CODE_NAME=focal
    ARG ARROW_VERSION_RULE="<2.0.0"
    FROM --build-arg CODE_NAME="$CODE_NAME" \
        --build-arg PYTHON_VERSION="$PYTHON_VERSION" \
        --build-arg ARROW_VERSION_RULE="$ARROW_VERSION_RULE" \
        +build

    ENTRYPOINT ["/bin/bash"]
    SAVE IMAGE turbodbc:latest

docker-compose:
    FROM +driver
    COPY . /src

    ENV ODBCSYSINI=/src/travis/odbc
    ENV TURBODBC_TEST_CONFIGURATION_FILES="query_fixtures_postgresql.json"
    WORKDIR /src

    WITH DOCKER --compose earthly-docker-compose.yml
        RUN /bin/bash -c "(r=10;while ! pg_isready --host=localhost --port=5432 --username=postgres ; do ((--r)) || exit; sleep 1 ;done)" && \
            echo "dsf"
    END
