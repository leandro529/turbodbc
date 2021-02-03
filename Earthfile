FROM ubuntu:focal
ENV DEBIAN_FRONTEND=noninteractive

RUN apt-get update && apt-get upgrade -y && \
    apt-get install -y unixodbc unixodbc-dev libboost-all-dev python3.8-dev python3.8-venv cmake g++ wget \
                       odbc-postgresql ninja-build gnupg apt-transport-https && \
    apt-get clean

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
    ACCEPT_EULA=Y apt-get install msodbcsql17 && \
    odbcinst -i -d -f /opt/microsoft/msodbcsql17/etc/odbcinst.ini

# install the exasol odbc driver:
RUN cd /opt && \
    wget -q https://www.exasol.com/support/secure/attachment/111075/EXASOL_ODBC-6.2.9.tar.gz && \
    tar xzf EXASOL_ODBC-6.2.9.tar.gz && \
    echo "\n[EXASOL]\nDriver=`ls /opt/EXA*/lib/linux/x86_64/libexaodbc-uo2214lv1.so`\nThreading=2\n" >> /etc/odbcinst.ini && \
    rm /opt/*.tar.gz

RUN cd /opt && \
    wget -q https://github.com/pybind/pybind11/archive/v2.6.2.tar.gz && \
    tar xvf v2.6.2.tar.gz && \
    rm /opt/*.tar.gz

# create a python virtualenv at /venv that contains the necessary build and test packages:
RUN python3.8 -m venv /venv && \
    /venv/bin/pip install numpy==1.20.0 pandas==1.2.1 'pyarrow<2' pybind11==2.6.2 pytest mock pytest-cov

# set locale to utf-8, which is required for some odbc drivers (mysql);
# also, set environment as set after 'source /venv/bin/activate'
ENV LC_ALL=C.UTF-8
ENV VIRTUAL_ENV=/venv
ENV PATH=/venv/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin

RUN pyarrow_path=`python -c 'import pyarrow; print(pyarrow.__path__[0])'` && \
    pyarrow_so_version=`python -c 'import pyarrow; print(pyarrow.cpp_build_info.so_version)'` && \
    ln -s $pyarrow_path/libarrow.so.${pyarrow_so_version} $pyarrow_path/libarrow.so && \
    ln -s $pyarrow_path/libarrow_python.so.${pyarrow_so_version} $pyarrow_path/libarrow_python.so

build:
    COPY . /src
    RUN mkdir /src/build
    WORKDIR /src/build
    RUN ln -s /opt/pybind11-2.6.2 /src/pybind11
    RUN cmake -DBUILD_COVERAGE=ON -DCMAKE_INSTALL_PREFIX=./dist -DPYBIND11_PYTHON_VERSION=3.8 -DDISABLE_CXX11_ABI=ON ..
    RUN make -j4
    RUN cmake --build . --target install
    SAVE ARTIFACT /src/build /build

test:
    COPY . /src
    COPY +build/build /src/build
    WORKDIR /src/build
    RUN ctest --verbose

docker:
    COPY . /src
    COPY +build/build /src/build
    ENTRYPOINT ["/bin/bash"]
    SAVE IMAGE turbodbc:latest
