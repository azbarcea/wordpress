FROM php:7.2-alpine

# install the PHP extensions we need
RUN set -ex; \
	\
	apk add --no-cache --virtual .build-deps \
		libjpeg-turbo-dev \
		libpng-dev \
	; \
	\
	docker-php-ext-configure gd --with-png-dir=/usr --with-jpeg-dir=/usr; \
	docker-php-ext-install gd mysqli opcache zip; \
	\
	runDeps="$( \
		scanelf --needed --nobanner --format '%n#p' --recursive /usr/local/lib/php/extensions \
			| tr ',' '\n' \
			| sort -u \
			| awk 'system("[ -e /usr/local/lib/" $1 " ]") == 0 { next } { print "so:" $1 }' \
	)"; \
	apk add --virtual .wordpress-phpexts-rundeps $runDeps; \
	apk del .build-deps

# set recommended PHP.ini settings
# see https://secure.php.net/manual/en/opcache.installation.php
RUN { \
		echo 'opcache.memory_consumption=128'; \
		echo 'opcache.interned_strings_buffer=8'; \
		echo 'opcache.max_accelerated_files=4000'; \
		echo 'opcache.revalidate_freq=2'; \
		echo 'opcache.fast_shutdown=1'; \
		echo 'opcache.enable_cli=1'; \
	} > /usr/local/etc/php/conf.d/opcache-recommended.ini

# install wp-cli dependencies
RUN apk add --no-cache \
# bash is needed for 'wp shell': https://github.com/wp-cli/shell-command/blob/b8dafcc2a2eba5732fdee70be077675a302848e9/src/WP_CLI/REPL.php#L104
		bash \
		less \
		mysql-client

RUN set -ex; \
	mkdir -p /var/www/html; \
	chown -R www-data:www-data /var/www/html
WORKDIR /var/www/html
VOLUME /var/www/html

# https://make.wordpress.org/cli/2018/05/31/gpg-signature-change/
# pub   rsa2048 2018-05-31 [SC]
#       63AF 7AA1 5067 C056 16FD  DD88 A3A2 E8F2 26F0 BC06
# uid           [ unknown] WP-CLI Releases <releases@wp-cli.org>
# sub   rsa2048 2018-05-31 [E]
ENV WORDPRESS_CLI_GPG_KEY 63AF7AA15067C05616FDDD88A3A2E8F226F0BC06

ENV WORDPRESS_CLI_VERSION 2.0.1
ENV WORDPRESS_CLI_SHA512 21b9c1d65993f88bf81cc73c0a832532cc424bea8c15563a77af1905d0dc4714f2af679dfadedd3b683f3968902b4b6be4c6cf94285da9f5582b30c1dac5397f

RUN set -ex; \
	\
	apk add --no-cache --virtual .fetch-deps \
		gnupg \
	; \
	\
	curl -o /usr/local/bin/wp.gpg -fSL "https://github.com/wp-cli/wp-cli/releases/download/v${WORDPRESS_CLI_VERSION}/wp-cli-${WORDPRESS_CLI_VERSION}.phar.gpg"; \
	\
	export GNUPGHOME="$(mktemp -d)"; \
	gpg --keyserver ha.pool.sks-keyservers.net --recv-keys "$WORDPRESS_CLI_GPG_KEY"; \
	gpg --batch --decrypt --output /usr/local/bin/wp /usr/local/bin/wp.gpg; \
	command -v gpgconf && gpgconf --kill all || :; \
	rm -rf "$GNUPGHOME" /usr/local/bin/wp.gpg; \
	\
	echo "$WORDPRESS_CLI_SHA512 */usr/local/bin/wp" | sha512sum -c -; \
	chmod +x /usr/local/bin/wp; \
	\
	apk del .fetch-deps; \
	\
	wp --allow-root --version

COPY docker-entrypoint.sh /usr/local/bin/

ENTRYPOINT ["docker-entrypoint.sh"]
USER www-data
CMD ["wp", "shell"]
