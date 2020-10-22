FROM php:7.3-fpm

RUN docker-php-ext-install pdo pdo_mysql

RUN apt-get update && apt-get install -y cron

# Add crontab file in the cron directory
ADD docker/crontab /etc/cron.d/cron

# Give execution rights on the cron job
RUN chmod 0644 /etc/cron.d/cron

# Create the log file to be able to run tail
RUN touch /var/log/cron.log

# Run the command on container startup
CMD cron && docker-php-entrypoint php-fpm
