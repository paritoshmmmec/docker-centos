# The official Canonical Ubuntu Bionic image is ideal from a security perspective,
# especially for the enterprises that we, the RabbitMQ team, have to deal with
FROM centos:7.5.1804

ENV GOSU_VERSION=1.11
RUN gpg --keyserver ha.pool.sks-keyservers.net --recv-keys B42F6819007F00F88E364FD4036A9C25BF357DD4 \
	&& curl -o /usr/local/bin/gosu -SL "https://github.com/tianon/gosu/releases/download/${GOSU_VERSION}/gosu-amd64" \
	&& curl -o /usr/local/bin/gosu.asc -SL "https://github.com/tianon/gosu/releases/download/${GOSU_VERSION}/gosu-amd64.asc" \
	&& gpg --verify /usr/local/bin/gosu.asc \
	&& rm /usr/local/bin/gosu.asc \
	&& rm -r /root/.gnupg/ \
	&& chmod +x /usr/local/bin/gosu \
	# Verify that the binary works
	&& gosu nobody true


RUN yum -y install epel-release \
	&& yum -y update \
	&& yum -y install pwgen wget logrotate python \
	&& yum -y install nss_wrapper gettext \
	&& wget https://packages.erlang-solutions.com/erlang-solutions-2.0-1.noarch.rpm \
	&& rpm -Uvh erlang-solutions-2.0-1.noarch.rpm \
	&& yum -y install erlang socat \
	&& yum clean all 


ENV RABBITMQ_DATA_DIR=/var/lib/rabbitmq
# Create rabbitmq system user & group, fix permissions & allow root user to connect to the RabbitMQ Erlang VM
RUN set -eux; \
	groupadd --gid 1001 --system rabbitmq; \
	useradd --uid 1001 --system --home-dir "$RABBITMQ_DATA_DIR" --gid rabbitmq rabbitmq; \
	mkdir -p "$RABBITMQ_DATA_DIR" /etc/rabbitmq /tmp/rabbitmq-ssl /var/log/rabbitmq; \
	chown -fR rabbitmq:rabbitmq "$RABBITMQ_DATA_DIR" /etc/rabbitmq /tmp/rabbitmq-ssl /var/log/rabbitmq; \
	chmod 777 "$RABBITMQ_DATA_DIR" /etc/rabbitmq /tmp/rabbitmq-ssl /var/log/rabbitmq; \
	ln -sf "$RABBITMQ_DATA_DIR/.erlang.cookie" /root/.erlang.cookie

# Use the latest stable RabbitMQ release (https://www.rabbitmq.com/download.html)
ENV RABBITMQ_VERSION 3.8.2
# https://www.rabbitmq.com/signatures.html#importing-gpg
ENV RABBITMQ_PGP_KEY_ID="0x0A9AF2115F4687BD29803A206B73A36E6026DFCA"
ENV RABBITMQ_HOME=/opt/rabbitmq

# Add RabbitMQ to PATH, send all logs to TTY
ENV PATH=$RABBITMQ_HOME/sbin:$PATH \
	RABBITMQ_LOGS=-

# Install RabbitMQ
RUN set -eux; \
	\
	yum -y update; \
	yum -y install epel-release \
	ca-certificates \
	gnupg \
	wget \
	xz-utils \
	; \
	rm -rf /var/lib/apt/lists/*; \
	\
	RABBITMQ_SOURCE_URL="https://github.com/rabbitmq/rabbitmq-server/releases/download/v$RABBITMQ_VERSION/rabbitmq-server-generic-unix-latest-toolchain-$RABBITMQ_VERSION.tar.xz"; \
	RABBITMQ_PATH="/usr/local/src/rabbitmq-$RABBITMQ_VERSION"; \
	\
	wget --progress dot:giga --output-document "$RABBITMQ_PATH.tar.xz.asc" "$RABBITMQ_SOURCE_URL.asc"; \
	wget --progress dot:giga --output-document "$RABBITMQ_PATH.tar.xz" "$RABBITMQ_SOURCE_URL"; \
	\
	export GNUPGHOME="$(mktemp -d)"; \
	gpg --batch --keyserver hkps://keys.openpgp.org --recv-keys "$RABBITMQ_PGP_KEY_ID"; \
	gpg --batch --verify "$RABBITMQ_PATH.tar.xz.asc" "$RABBITMQ_PATH.tar.xz"; \
	echo erl --version \
	rm -rf "$GNUPGHOME"; \
	\
	mkdir -p "$RABBITMQ_HOME"; \
	tar --extract --file "$RABBITMQ_PATH.tar.xz" --directory "$RABBITMQ_HOME" --strip-components 1; \
	rm -rf "$RABBITMQ_PATH"*; \
	# Do not default SYS_PREFIX to RABBITMQ_HOME, leave it empty
	grep -qE '^SYS_PREFIX=\$\{RABBITMQ_HOME\}$' "$RABBITMQ_HOME/sbin/rabbitmq-defaults"; \
	sed -i 's/^SYS_PREFIX=.*$/SYS_PREFIX=/' "$RABBITMQ_HOME/sbin/rabbitmq-defaults"; \
	grep -qE '^SYS_PREFIX=$' "$RABBITMQ_HOME/sbin/rabbitmq-defaults"; \
	chown -R rabbitmq:rabbitmq "$RABBITMQ_HOME"; \
	# verify assumption of no stale cookies
	[ ! -e "$RABBITMQ_DATA_DIR/.erlang.cookie" ]; \
	# Ensure RabbitMQ was installed correctly by running a few commands that do not depend on a running server, as the rabbitmq user
	# If they all succeed, it's safe to assume that things have been set up correctly
	gosu rabbitmq rabbitmqctl help; \
	gosu rabbitmq rabbitmqctl list_ciphers; \
	gosu rabbitmq rabbitmq-plugins list; \
	yum clean all; \
	# no stale cookies
	rm "$RABBITMQ_DATA_DIR/.erlang.cookie" 


# Added for backwards compatibility - users can simply COPY custom plugins to /plugins
RUN ln -sf /opt/rabbitmq/plugins /plugins

RUN rabbitmq-plugins enable --offline rabbitmq_management

# set home so that any `--user` knows where to put the erlang cookie
ENV HOME $RABBITMQ_DATA_DIR
# Hint that the data (a.k.a. home dir) dir should be separate volume
VOLUME $RABBITMQ_DATA_DIR

# warning: the VM is running with native name encoding of latin1 which may cause Elixir to malfunction as it expects utf8. Please ensure your locale is set to UTF-8 (which can be verified by running "locale" in your shell)
# Setting all environment variables that control language preferences, behaviour differs - https://www.gnu.org/software/gettext/manual/html_node/The-LANGUAGE-variable.html#The-LANGUAGE-variable
# https://docs.docker.com/samples/library/ubuntu/#locales
ENV LANG=en_US.UTF-8 LANGUAGE=en_US.UTF-8 LC_ALL=en_US.UTF-8

COPY docker-entrypoint.sh /usr/local/bin/
ENTRYPOINT ["docker-entrypoint.sh"]

EXPOSE 4369 5671 5672 25672
CMD ["rabbitmq-server"]
