FROM debian:bookworm-slim

ENV DEBIAN_FRONTEND=noninteractive

RUN set -eux; \
    echo 'postfix postfix/mailname string lab.test' | debconf-set-selections; \
    echo 'postfix postfix/main_mailer_type select Internet Site' | debconf-set-selections; \
    apt-get update; \
    apt-get install -y --no-install-recommends postfix ca-certificates; \
    useradd -m -s /usr/sbin/nologin student; \
    rm -rf /var/lib/apt/lists/*

ARG MAIL_NODE

COPY config/mail-${MAIL_NODE}/main.cf /etc/postfix/main.cf

RUN set -eux; \
    chown root:root /etc/postfix/main.cf; \
    chmod 0644 /etc/postfix/main.cf; \
    postconf -F 'smtp/unix/chroot=n'

EXPOSE 25

CMD ["postfix", "start-fg"]

