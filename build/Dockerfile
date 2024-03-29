FROM debian:jessie

RUN rm /etc/apt/sources.list
RUN echo "deb http://archive.debian.org/debian-security jessie/updates main" >> /etc/apt/sources.list.d/jessie.list
RUN echo "deb http://archive.debian.org/debian jessie main" >> /etc/apt/sources.list.d/jessie.list



# Let the container know that there is no tty
ENV DEBIAN_FRONTEND noninteractive

# --- set timezone
ARG BUILD_TZ="Europe/Berlin"
RUN echo "$BUILD_TZ" > /etc/timezone && \
    dpkg-reconfigure -f noninteractive tzdata


# install packages
RUN apt-get update && \
apt-get --force-yes -y install rsync nano cron && \
apt-get --force-yes -y install supervisor && \
apt-get --force-yes -y install nginx && \
apt-get --force-yes -y install php5-fpm && \
apt-get --force-yes -y install php5-mysql && \
apt-get --force-yes -y install php5-mcrypt && \
apt-get --force-yes -y install php5-xdebug && \
apt-get --force-yes -y install php5-mysqlnd && \
apt-get --force-yes -y install php5-curl && \
apt-get --force-yes -y install php-mail && \
apt-get --force-yes -y install php5-json && \
apt-get --force-yes -y install phpunit && \
apt-get --force-yes -y install sendemail && \
apt-get --force-yes -y install openssh-server && \
apt-get --force-yes -y install openssh-client


ARG BUILD_PRINTING="no"
RUN if [ "$BUILD_PRINTING" = "yes" ] ; then  apt-get --force-yes install -y cups-client cups-bsd ; fi ;
ARG BUILD_LATEX="no"
#RUN if [ "$BUILD_LATEX" = "yes" ] ;    then  apt-get install -y libdmtx-utils dblatex latex-make texlive-lang-german ; fi ;
RUN if [ "$BUILD_LATEX" = "yes" ] ;    then  apt-get --force-yes install -y libdmtx-utils texlive-base texlive-lang-german texlive-latex-extra ; fi ;
ARG BUILD_MYSQLCLI="no"
RUN if [ "$BUILD_MYSQLCLI" = "yes" ] ; then  apt-get --force-yes install -y mysql-client ; fi ;


RUN apt-get --purge remove -y tex.\*-doc$ cpp cpp-4.9

RUN apt-get clean && apt-get -y autoremove  && rm -rf /var/lib/apt/lists/*

RUN rm -rf /usr/share/doc-base /usr/shre/doc /usr/share/man


RUN touch /var/log/cron.log


# prepare nginx
RUN sed -i -e"s/worker_processes  1/worker_processes 5/" /etc/nginx/nginx.conf && \
sed -i -e"s/keepalive_timeout\s*65/keepalive_timeout 2/" /etc/nginx/nginx.conf && \
sed -i -e"s/keepalive_timeout 2/keepalive_timeout 2;\n\tclient_max_body_size 100m/" /etc/nginx/nginx.conf && \
echo "daemon off;" >> /etc/nginx/nginx.conf

# prepare php-fpm
RUN  sed -i -e "s/;cgi.fix_pathinfo=1/cgi.fix_pathinfo=0/g" /etc/php5/fpm/php.ini && \
sed -i -e "s/upload_max_filesize\s*=\s*2M/upload_max_filesize = 100M/g" /etc/php5/fpm/php.ini && \
sed -i -e "s/post_max_size\s*=\s*8M/post_max_size = 100M/g" /etc/php5/fpm/php.ini && \
sed -i -e "s/;daemonize\s*=\s*yes/daemonize = no/g" /etc/php5/fpm/php-fpm.conf && \
sed -i -e "s/;catch_workers_output\s*=\s*yes/catch_workers_output = yes/g" /etc/php5/fpm/pool.d/www.conf && \
sed -i -e "s/pm.max_children = 5/pm.max_children = 9/g" /etc/php5/fpm/pool.d/www.conf && \
sed -i -e "s/pm.start_servers = 2/pm.start_servers = 3/g" /etc/php5/fpm/pool.d/www.conf && \
sed -i -e "s/pm.min_spare_servers = 1/pm.min_spare_servers = 2/g" /etc/php5/fpm/pool.d/www.conf && \
sed -i -e "s/pm.max_spare_servers = 3/pm.max_spare_servers = 4/g" /etc/php5/fpm/pool.d/www.conf && \
sed -i -e "s/pm.max_requests = 500/pm.max_requests = 200/g" /etc/php5/fpm/pool.d/www.conf

# prepare ownership of php-fpm socket file
RUN sed -i -e "s/;listen.mode = 0660/listen.mode = 0750/g" /etc/php5/fpm/pool.d/www.conf && \
find /etc/php5/cli/conf.d/ -name "*.ini" -exec sed -i -re 's/^(\s*)#(.*)/\1;\2/g' {} \;

# mycrypt conf
RUN php5enmod mcrypt

# nginx site conf
RUN rm -Rf /etc/nginx/conf.d/* && \
rm -Rf /etc/nginx/sites-enabled/default && \
rm -Rf /etc/nginx/sites-available/default && \
mkdir -p /etc/nginx/ssl/
ADD ./default /etc/nginx/sites-available/default
RUN ln -s /etc/nginx/sites-available/default /etc/nginx/sites-enabled/default
ADD ./index.php /usr/share/nginx/html/index.php
RUN chown -Rf www-data.www-data /usr/share/nginx/html/

# Start Supervisord
ADD ./start.sh /start.sh
RUN chmod 755 /start.sh

#
# establisch supervisord
#
ADD /supervisor /etc/supervisor
# link old /etc/init.d/ startup scripts to supervisor
RUN ls -m1    /etc/supervisor/services.d | while read i; do mv /etc/init.d/$i /etc/init.d/$i-orig ;  ln -sf /etc/supervisor/super-init.sh /etc/init.d/$i ; done
RUN ln -sf    /etc/supervisor/systemctl /bin/systemctl
RUN chmod a+x /etc/supervisor/* /etc/supervisor/*.d/*
copy /supervisor/invoke-rc.d /usr/sbin/invoke-rc.d 
#
# create directory for service volume
#
RUN mkdir -p /service ; chmod a+rwx /service




############################################################################################
#
# docker-extensions
#
RUN mkdir -p  /usr/local/bin
COPY ./bin/*  /usr/local/bin/
RUN chmod a+x /usr/local/bin/*




# forward request and error logs to docker log collector
RUN ln -sf /dev/stdout /var/log/nginx/access.log \
	&& ln -sf /dev/stderr /var/log/nginx/error.log

EXPOSE 80 443

ENV TERM xterm
#
# startup script
#
ADD ./start.sh /start.sh
RUN chmod 755 /start.sh
CMD ["/start.sh"]
