FROM php:5.6-apache
MAINTAINER Vaclav Adamec <vaclav.adamec@suchy-zleb.cz>

ENV WORDPRESS_VERSION 4.7.2
ENV WORDPRESS_SHA1 7b687f1af589c337124e6247229af209ec1d52c3

RUN set -ex; \
	curl -o wordpress.tar.gz -fSL "https://wordpress.org/wordpress-${WORDPRESS_VERSION}.tar.gz"; \
	echo "$WORDPRESS_SHA1 *wordpress.tar.gz" | sha1sum -c -; \
# upstream tarballs include ./wordpress/ so this gives us /usr/src/wordpress
	tar -xzf wordpress.tar.gz -C /usr/src/; \
	rm wordpress.tar.gz; \
	chown -R www-data:www-data /usr/src/wordpress

# install the PHP extensions we need
RUN set -ex; \
	docker-php-ext-install mysqli

COPY docker-entrypoint.sh /usr/local/bin/

VOLUME ["/var/www/html"]

ENTRYPOINT ["docker-entrypoint.sh"]

CMD ["cat"]
