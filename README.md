credits: nimmis:docker-debian

## Debian Wheezy/Jessie version
 [![Docker Hub; huxy/docker-debian](https://img.shields.io/badge/dockerhub-docker--debian-blue.svg)](https://registry.hub.docker.com/h/huxy/docker-debian)

This is a docker base image using Debian. It's forked from nimmis:docker-debian but has been further streamlined. 
It is provided with a working init process, syslog and a cron daemon. 

### Why use this image

The unix process ID 1 is the process to receive the SIGTERM signal when you execute a 

	docker stop <container ID>

if the container has the command `CMD ["bash"]` then bash process will get the SIGTERM signal and terminate.
All other processes running on the system will just stop without the possibility to shutdown correclty

### my_init init script

The container has a script that handles the init process using the [supervisor system](http://supervisord.org/index.html) to start
the daemons to run and also catch signals (SIGTERM) to shutdown all processes started by supervisord. The script is a modification of
the script made by Phusion, hence supervisor and not runit. There are also two directories to run scripts before any daemon is started.

#### Run script once /etc/my_runonce

All executables in this directory are run at start, after completion the script is removed from the directory

#### Run script every start /etc/my_runalways

All executable in this directory are run at every start of the container, ie, at `docker run` and `docker start`

#### Permanent output to docker log when starting container

Each time the container is started the content of the file /tmp/startup.log is displayed so if your startup scripts generate 
vital information to be shown please add that information to that file. This information can be retrieved anytime by
executing `docker logs <container id>`

### cron daemon

This image provides a running cron daemon. Cronjobs themselves do not automatically import environment varibles. You can either 
import variables using env >> /etc/environment or sourcing the container_environment.sh file as part of a bash script. 

### syslog-ng

No all services works without a syslog daemon, if you don't have one running those messages is lost in space,
all messages sent via the syslog daemon is saved in /var/log/syslog

### Docker fixes 

Also there are fixed (besideds the init process) assosiated with running Debian inside a docker container.

### New commands autostarted by supervisord

To add other processes to run automaticly, add a file ending with .conf  in /etc/supervisor/conf.d/ 
with a layout like this (/etc/supervisor/conf.d/myprogram.conf) 

	[program:myprogram]
	command=/usr/bin/myprogram

`myprogram` is the name of this process when working with supervisctl.

Output logs std and error is found in /var/log/supervisor/ and the files begins with the <defined name><-stdout|-stderr>superervisor*.log

For more settings please consult the [manual FOR supervisor](http://supervisord.org/configuration.html#program-x-section-settings)

#### starting commands from /etc/init.d/ or commands that detach with my_service

The supervisor process assumes that a command that ends has stopped so if the command detach it will try to restart it. To work around this
problem I have written an extra command to be used for these commands. First you have to make a normal start/stop command and place it in
the /etc/init.d that starts the program with

	/etc/init.d/command start or
	service command start

and stops with

        /etc/init.d/command stop or
        service command stop

Configure the configure-file (/etc/supervisor/conf.d/myprogram.conf)

	[program:myprogram]
	command=/my_service myprogram

There is an optional parameter, to run a script after a service has start, e.g to run the script /usr/local/bin/postproc.sh av myprogram is started

        [program:myprogram]
        command=/my_service myprogram /usr/local/bin/postproc.sh

### Output information to docker logs

The console output is owned by the my_init process so any output from commands woun't show in the docker log. To send a text from any command, either
at startup och during run, append the output to the file /var/log/startup.log, e.g sending specific text to log

	echo "Application is finished" >> /var/log/startup.log

or output from script

	/usr/local/bin/myscript >> /var/log/startlog.log


	> docker run -d --name debian nimmis/debian
	> docker logs debian
	*** open logfile
	*** Run files in /etc/my_runonce/
	*** Run files in /etc/my_runalways/
	*** Running /etc/rc.local...
	*** Booting supervisor daemon...
	*** Supervisor started as PID 9
	2015-08-04 11:34:06,763 CRIT Set uid to user 0
	*** Started processes via Supervisor......
	crond                            RUNNING    pid 13, uptime 0:00:04
	syslog-ng                        RUNNING    pid 12, uptime 0:00:04

	> docker exec debian sh -c 'echo "Testmessage to log" >> /var/log/startup.log'
	> docker logs debian
        *** open logfile
        *** Run files in /etc/my_runonce/
        *** Run files in /etc/my_runalways/
        *** Running /etc/rc.local...
        *** Booting supervisor daemon...
        *** Supervisor started as PID 9
        2015-08-04 11:34:06,763 CRIT Set uid to user 0
        *** Started processes via Supervisor......
        crond                            RUNNING    pid 13, uptime 0:00:04
        syslog-ng                        RUNNING    pid 12, uptime 0:00:04

	*** Log: Testmessage to log
        >
Environment variables

If you use /sbin/my_init as the main container command, then any environment variables set with docker run --env or with the ENV command in the Dockerfile, will be picked up by my_init. These variables will also be passed to all child processes, including /etc/my_init.d startup scripts, Runit and Runit-managed services. There are however a few caveats you should be aware of:

Environment variables on Unix are inherited on a per-process basis. This means that it is generally not possible for a child process to change the environment variables of other processes.
Because of the aforementioned point, there is no good central place for defining environment variables for all applications and services. Debian has the /etc/environment file but it only works in some situations.
Some services change environment variables for child processes. Nginx is one such example: it removes all environment variables unless you explicitly instruct it to retain them through the env configuration option. If you host any applications on Nginx (e.g. using the passenger-docker image, or using Phusion Passenger in your own image) then they will not see the environment variables that were originally passed by Docker.
my_init provides a solution for all these caveats.


Centrally defining your own environment variables

During startup, before running any startup scripts, my_init imports environment variables from the directory /etc/container_environment. This directory contains files who are named after the environment variable names. The file contents contain the environment variable values. This directory is therefore a good place to centrally define your own environment variables, which will be inherited by all startup scripts and Runit services.

For example, here's how you can define an environment variable from your Dockerfile:

RUN echo Apachai Hopachai > /etc/container_environment/MY_NAME
You can verify that it works, as follows:

$ docker run -t -i <YOUR_NAME_IMAGE> /sbin/my_init -- bash -l
...
*** Running bash -l...
# echo $MY_NAME
Apachai Hopachai
Handling newlines

If you've looked carefully, you'll notice that the 'echo' command actually prints a newline. Why does $MY_NAME not contain a newline then? It's because my_init strips the trailing newline, if any. If you intended on the value having a newline, you should add another newline, like this:

RUN echo -e "Apachai Hopachai\n" > /etc/container_environment/MY_NAME

Environment variable dumps

While the previously mentioned mechanism is good for centrally defining environment variables, it by itself does not prevent services (e.g. Nginx) from changing and resetting environment variables from child processes. However, the my_init mechanism does make it easy for you to query what the original environment variables are.

During startup, right after importing environment variables from /etc/container_environment, my_init will dump all its environment variables (that is, all variables imported from container_environment, as well as all variables it picked up from docker run --env) to the following locations, in the following formats:

/etc/container_environment
/etc/container_environment.sh - a dump of the environment variables in Bash format. You can source the file directly from a Bash shell script.
/etc/container_environment.json - a dump of the environment variables in JSON format.
The multiple formats makes it easy for you to query the original environment variables no matter which language your scripts/apps are written in.

Here is an example shell session showing you how the dumps look like:

$ docker run -t -i \
  --env FOO=bar --env HELLO='my beautiful world' \
  phusion/baseimage:<VERSION> /sbin/my_init -- \
  bash -l
...
*** Running bash -l...
# ls /etc/container_environment
FOO  HELLO  HOME  HOSTNAME  PATH  TERM  container
# cat /etc/container_environment/HELLO; echo
my beautiful world
# cat /etc/container_environment.json; echo
{"TERM": "xterm", "container": "lxc", "HOSTNAME": "f45449f06950", "HOME": "/root", "PATH": "/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin", "FOO": "bar", "HELLO": "my beautiful world"}
# source /etc/container_environment.sh
# echo $HELLO
my beautiful world

Modifying environment variables

It is even possible to modify the environment variables in my_init (and therefore the environment variables in all child processes that are spawned after that point in time), by altering the files in /etc/container_environment. After each time my_init runs a startup script, it resets its own environment variables to the state in /etc/container_environment, and re-dumps the new environment variables to container_environment.sh and container_environment.json.

But note that:

modifying container_environment.sh and container_environment.json has no effect.
Runit services cannot modify the environment like that. my_init only activates changes in /etc/container_environment when running startup scripts.

Security

Because environment variables can potentially contain sensitive information, /etc/container_environment and its Bash and JSON dumps are by default owned by root, and accessible only by the docker_env group (so that any user added this group will have these variables automatically loaded).

If you are sure that your environment variables don't contain sensitive data, then you can also relax the permissions on that directory and those files by making them world-readable:

RUN chmod 755 /etc/container_environment
RUN chmod 644 /etc/container_environment.sh /etc/container_environment.json

### Installation

This continer should normaly run as a daemon i.e with the `-d` flag attached

	docker run -d nimmis/debian

but if you want to check if all services has been started correctly you can start with the following command

	docker run -ti nimmis/debian

the output, if working correctly should be

	docker run -ti nimmis/debian
	*** open logfile
	*** Run files in /etc/my_runonce/
	*** Run files in /etc/my_runalways/
	*** Running /etc/rc.local...
	*** Booting supervisor daemon...
	*** Supervisor started as PID 11
	2015-01-02 10:45:43,750 CRIT Set uid to user 0
	*** Started processes via Supervisor......
	crond                            RUNNING    pid 15, uptime 0:00:02
	syslog-ng                        RUNNING    pid 14, uptime 0:00:02

pressing a CTRL-C in that window  or running `docker stop <container ID>` will generate the following output

	*** Shutting down supervisor daemon (PID 11)...
	*** Killing all processes...

you can the restart that container with 

	docker start <container ID>

Accessing the container with a bash shell can be done with

	docker exec -ti <container ID> /bin/bash


