FROM ubuntu:latest

RUN apt-get update -q && apt-get install -qy python-pip
RUN pip install elasticsearch-curator==5.8.1
RUN apt-get install -qy cron && apt-get -qy install vim