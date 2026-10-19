# Postfix SMTP Lab

A collection of hands-on laboratories for learning how Postfix and SMTP actually work.

The goal is to understand mail delivery, SMTP responses, queues, retries, and error handling through reproducible experiments rather than configuration examples alone.

Each lab runs in an isolated Docker Compose environment and focuses on a specific aspect of mail infrastructure.

## Available labs

### [01 — Basic SMTP](labs/01-basic-smtp/)

Two Postfix servers communicating over SMTP.

Learn how to:

- Send messages between two Postfix servers.
- Understand the roles of `smtpd`, `smtp`, `qmgr`, and `local`.
- Inspect mail queues and delivery logs.
- Simulate an unavailable destination server.
- Observe deferred delivery and retry behavior.
- Distinguish temporary delivery failures from permanent errors.
- Understand when responsibility for a message transfers between SMTP servers.

## Quick start

Requirements:

- Docker Engine
- Docker Compose plugin

Clone the repository and start the first lab:

```bash
git clone https://github.com/shiver25/postfix-smtp-lab
cd postfix-smtp-lab/labs/01-basic-smtp

docker compose up -d --build
docker compose ps
```

Follow the experiments described in the [lab README](labs/01-basic-smtp/README.md).

Stop the lab:

```bash
docker compose down
```

Named volumes are preserved by default. Running `docker compose down -v` also removes the lab's mail queues and stored messages.

## Design principles

- **Learn by experimenting.** Each lab demonstrates observable SMTP and Postfix behavior.
- **Keep it reproducible.** Labs use Docker Compose and explicit configuration files.
- **Keep it isolated.** No SMTP ports are published to the host, and the first lab uses an internal Docker network.
- **Explain failures.** SMTP response codes, enhanced status codes, logs, and queue states are part of the learning process.
- **One concept at a time.** New components are introduced only when needed.

## Security notice

These laboratories are intended for local learning and testing.

They are not production-ready mail servers. In particular, the first lab does not implement TLS, SMTP authentication, spam filtering, or a complete mail delivery system.

Do not expose the lab containers directly to the Internet.

## License

License information will be added before publication.
