FROM debian:wheezy

MAINTAINER nimmis <kjell.havneskold@gmail.com>

# disable interactive functions
ENV DEBIAN_FRONTEND noninteractive

# Do OS fixes
ADD bin/fix.sh /fix.sh
#RUN chmod +x /fix.sh && /fix.sh && rm /fix.sh
RUN /bin/bash /fix.sh && rm -f /fix.sh

# configure supervisor
ADD ./supervisor/supervisord.conf /etc/supervisor/
ADD ./supervisor/conf.d/crond.conf /etc/supervisor/conf.d/
ADD ./supervisor/conf.d/syslog-ng.conf /etc/supervisor/conf.d/

# Add my_init script and help scripts 
ADD bin/my_init /
ADD bin/my_service /
RUN chmod +x /my_*

# Set environment variables.
ENV HOME /root

# Define working directory.
WORKDIR /root

# Define default command.
CMD ["/my_init"]
