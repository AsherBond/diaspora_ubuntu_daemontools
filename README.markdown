[durl]:http://joindiaspora.com/ "Diaspora"
[daurl]:http://cr.yp.to/daemontools.html "Daemontools"
[dturl]:http://packages.ubuntu.com/da/maverick/daemontools
[drurl]:http://packages.ubuntu.com/da/maverick/daemontools-run
# [Daemontools][daurl] for [Diaspora\*][durl], on Ubuntu

## Rationale

It seems that [Diaspora\*][durl] is very picky about which services start
first. This is my attempt to decouple from the monolithic ./script/server,
which provides no recourse for services which disappear. As best I can tell,
the order of services should be (mongo|mysql, nginx, redis, resque-web),
websocket, resque-worker, and thin.  The websocket run script will wait for
redis availability. The resque worker will wait for websocket, and thin will
wait for mongo|mysql, nginx, redis, and websocket availability.

If a script fails to connect to a dependent service 10 times, the run script
will exit and try again. This allows you to monitor `svstat` for short lived
processes. Each time the run script attempts to connect to a dependent service,
it will be echoed to STDOUT. This will allow you to monitor readproctitle for
errant services as well.

## Requirements

+ Ubuntu 10.10 (the only version I tested with)
+ [Diaspora\*][durl] requirements
    + As of this writing, it includes
        + MTA (we run postfix)
        + Web server (we run nginx)
        + Redis
        + MongoDB|MySQL
        + Debian packages: [daemontools][dturl], [daemontools-run][drurl]

## Installation

1. Get your packages (pre MySQL conversion)
    * `apt-get install postfix nginx redis-server mongo daemontools daemontools-run`
    1. Stop init from starting your [Diaspora\*][durl] services
        * `update-rc.d -f mongodb remove`
        * `update-rc.d -f nginx remove`
        * `update-rc.d -f redis-server remove`
    1. Stop services
        * `service mongodb stop`
        * `service nginx stop`
        * `service redis-server stop`
1. Get your packages (MySQL configuration)
    * `apt-get install postfix nginx redis-server mysql-server mysql-client libmysqlclient-dev daemontools daemontools-run`
    1. Stop upstart/init from starting your [Diaspora\*][durl] services
        * `$EDITOR /etc/init/mysql.conf`
				    * `start on runlevel [!0123456]`
						* `stop on runlevel [!0123456]`
        * `update-rc.d -f nginx remove`
        * `update-rc.d -f redis-server remove`
    1. Stop services
        * `service mysql stop`
        * `service nginx stop`
        * `service redis-server stop`
1. Disable nginx daemonizing (ie `daemon off;`)
    * `$EDITOR /etc/nginx/nginx.conf`
		* `daemon off;`
1. Disable redis daemonizing (ie `daemonize no`)
    * `$EDITOR /etc/redis.conf`
		* `daemonize no`
1. Copy the etc skeleton into place
`cd [GIT_SRC_TREE]`
    * `find etc usr | cpio -dpm /`
    * `chmod a+x /etc/service/*/run /usr/local/bin/diaspora_svc.sh`
1. Edit the defaults if you need to
    * `$EDITOR /etc/default/diaspora`
1. Double check these scripts do what you expect! Our system currently runs three individual thins, over UNIX domain sockets. If your nginx isn't expecting that, you should probably update the thin run scripts.
    * `rm -rf /etc/service/thin_[12]`
    * `$EDITOR /etc/service/99thin_0/run`
1. Start the daemontools service scanner (should run automatically, during next boot)
    * `initctl start svscan`

## Customization

1. Change the default db store
    * `$EDITOR /etc/default/diaspora`
		* `DB_SERVICE=NAME` # mysql or mongo
1. Adding new thin processes
    * `mkdir /etc/service/99thin_N`
    * `cp /etc/service/99thin_0/run /etc/service/99thin_N`
1. Set thin user and group in /etc/default/diaspora
    * `THIN_USER=nobody`
    * `THIN_GROUP=nogroup`
1. Set rails environment in /etc/default/diaspora
    * `RAILS_ENV=development`
1. Set diaspora root directory in /etc/default/diaspora
    * `DIASPORA_HOME=/usr/local/app/diaspora`
1. Globally define the max attempts scripts will try to connect to services
    * `DEFAULT_ATTEMPTS=10`
1. Modify a single service's max attempts it will try to connect to a service
    * in a run script: `wait_service -a N`
1. Change the default resque web port
    * `$EDITOR /etc/default/diaspora`
		* `RESQUE_WEB_PORT=N`
		* `svc -i /etc/service/70resque_web`
1. Add resque\_web to nginx
    * See diaspora/chef/cookbooks/common/templates/default/nginx.conf.erb

## diaspora\_svc.sh usage

`/usr/local/bin/diaspora_svc.sh {start|stop|restart|status|startall|stopall|restartall|statusall}`

This script is by no means required, but there are some useful features to it.
It allows you to easily control diaspora related services (resque\_worker,
redis, thin, and websocket) and everything else controlled by svscan.

+ *start* allows you to start diaspora related services in /etc/service
+ *startall* starts all services in /etc/service
+ *stop* allows you to stop diaspora related services in /etc/service
+ *stopall* stops all services in /etc/service
+ *restart* allows you to restart diaspora related services in /etc/service
+ *restartall* restarts all services in /etc/service
+ *status* allows you to see status for diaspora related services in /etc/service
+ *statusall* prints all services in /etc/service

