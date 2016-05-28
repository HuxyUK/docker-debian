FROM debian:jessie

MAINTAINER nimmis <docker@gmail.com>

# disable interactive functions
# Set correct environment variables
ENV HOME /root
ENV DEBIAN_FRONTEND noninteractive
ENV LC_ALL C.UTF-8
ENV LANG en_US.UTF-8
ENV LANGUAGE en_US.UTF-8

# Exclude Documentation
ADD etc/dpkg/dpkg.cfg.d/01_nodoc /etc/dpkg/dpkg.cfg.d/01_nodoc

# Do OS fixes
ADD bin/setup.sh /setup.sh
RUN /bin/bash setup.sh && rm /setup.sh

# configure supervisor
ADD ./supervisor/supervisord.conf /etc/supervisor/
ADD ./supervisor/conf.d/crond.conf /etc/supervisor/conf.d/
ADD ./supervisor/conf.d/syslog-ng.conf /etc/supervisor/conf.d/

# Add my_init script and help scripts
ADD bin/my_init /
ADD bin/my_service /
RUN chmod +x /my_*

# Define working directory.
WORKDIR /root

# Disable SSH
RUN rm -rf /etc/service/sshd /etc/my_init.d/00_regen_ssh_host_keys.sh

# Define default command.
CMD ["/my_init"]



