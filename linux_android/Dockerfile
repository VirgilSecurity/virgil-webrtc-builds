FROM ubuntu:18.04
MAINTAINER Virgil Security Inc. <support@virgilsecurity.com>

RUN adduser -U tomcat
RUN usermod -aG sudo tomcat
USER tomcat

ARG DEBIAN_FRONTEND=noninteractive

# Set the locale
RUN apt-get clean
RUN apt-get -y update
RUN apt-get install -y locales
RUN locale-gen en_US.UTF-8
ENV LANG en_US.UTF-8
ENV LANGUAGE en_US:en
ENV LC_ALL en_US.UTF-8

RUN apt-get update -qq
RUN apt-get install -qq sudo curl git python lbzip2 lsb-release dialog apt-utils

#
#  WebRTC dependencies.
#
WORKDIR /tmp
RUN curl https://chromium.googlesource.com/chromium/src/+/master/build/install-build-deps-android.sh?format=TEXT | base64 -d > install-build-deps-android.sh
RUN curl https://chromium.googlesource.com/chromium/src/+/master/build/install-build-deps.sh?format=TEXT | base64 -d > install-build-deps.sh
RUN chmod +x ./install-build-deps.sh
RUN chmod +x ./install-build-deps-android.sh
RUN echo 'debconf debconf/frontend select Noninteractive' | debconf-set-selections
RUN ./install-build-deps.sh --no-chromeos-fonts
RUN ./install-build-deps-android.sh
