FROM debian:latest
MAINTAINER Matt Kereczman <kermat60@gmail.com>

RUN apt update &&  apt -y install curl

RUN curl -s https://packagecloud.io/install/repositories/akopytov/sysbench/script.deb.sh | bash
RUN apt -y install sysbench
