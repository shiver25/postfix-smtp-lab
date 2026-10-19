[English](README.md) | [Polski](README.pl.md)

# Postfix Lab — Lesson 1: Two Servers, Mail Queues, and SMTP

A hands-on laboratory for understanding how Postfix works. At this stage, **there is no Dovecot, TLS, SASL, or public SMTP access**.

We start with two mail servers and one fundamental question:

**Who holds a message when the destination server is unavailable?**

## Architecture

```text
 [mail-a] --SMTP TCP/25--> [mail-b] --> local(8) --> /var/mail/student
    |                         |
 queue-a                   queue-b

Docker network: mail-lab (internal: true)
No container ports are published to the host.
```

- **mail-a**: the sending server. `relayhost = [mail-b]:25` routes messages for domains outside `mydestination` to `mail-b`. The square brackets disable MX lookups for the next hop.
- **mail-b**: the receiving server. The domain `b.lab.test` belongs to `mydestination`. Messages for the local system user `student` are delivered using the `local(8)` transport — **not LMTP**.
- **Network**: Docker Compose uses `internal: true` to restrict external connectivity. Port 25 is not published to the host. Docker access from the host still exists, so containers should not be considered a security boundary for untrusted code.
- **Volumes**: separate named volumes preserve the queues of both Postfix instances (`queue-a`, `queue-b`) and the recipient's mailbox (`mailbox-b`) across normal container restarts.
- **Images**: Debian 12 (`bookworm-slim`) with the distribution's Postfix package, rather than an all-in-one mail server image.

## Project structure

```text
postfix-smtp-lab/
├── README.md
├── README.pl.md
├── LICENSE
└── labs/
    └── 01-basic-smtp/
        ├── README.md
        ├── README.pl.md
        ├── compose.yaml
        ├── Dockerfile
        └── config/
            ├── mail-a/main.cf
            └── mail-b/main.cf
```

## Requirements and quick start

You need Docker Engine with the Docker Compose plugin and Internet access to download the base image and packages **during the build**.

The internal Docker network applies to running containers, not to the image build process.

From the repository root:

```bash
cd labs/01-basic-smtp

docker compose config
docker compose up -d --build
docker compose ps
```

Inspect the effective Postfix configuration:

```bash
docker compose exec mail-a postconf myhostname mydestination relayhost
docker compose exec mail-b postconf myhostname mydestination default_transport
```

If startup fails with an error related to `maillog_file`, check the Postfix version and container logs:

```bash
docker compose exec mail-a postconf mail_version
docker compose logs mail-a mail-b
```

The laboratory relies on Postfix logging to standard output when running with `postfix start-fg`. This functionality may not be available in older Postfix releases.

## Experiment A: Successful message delivery

Send a message from `mail-a` using Postfix's local `sendmail` interface:

```bash
printf 'From: tester@a.lab.test\nTo: student@b.lab.test\nSubject: SMTP test\n\nHello from mail-a.\n' \
  | docker compose exec -T mail-a sendmail -i \
      -f tester@a.lab.test student@b.lab.test
```

Inspect both mail queues and the logs:

```bash
docker compose exec mail-a postqueue -p
docker compose exec mail-b postqueue -p
docker compose logs --tail=80 mail-a mail-b
```

Read the recipient's local mailbox:

```bash
docker compose exec mail-b \
  sh -c 'ls -l /var/mail/student; tail -n 40 /var/mail/student'
```

### What should happen?

The sending server delivers the message to `mail-b` over SMTP. The receiving server accepts it and delivers it to `/var/mail/student`.

A successful transfer was observed in our laboratory:

```text
mail-a:
E25FAA2C41: to=<student@b.lab.test>,
relay=mail-b[172.12.3.3]:25,
dsn=2.0.0, status=sent
(250 2.0.0 Ok: queued as ECEC88ED85)
```

The sender then removes the message from its queue:

```text
E25FAA2C41: removed
```

On `mail-b`:

```text
ECEC88ED85: to=<student@b.lab.test>,
relay=local, dsn=2.0.0,
status=sent (delivered to mailbox)
```

Queue IDs and container IP addresses will differ between runs.

Notice that each server assigns its own **queue ID**:

```text
mail-a: E25FAA2C41
mail-b: ECEC88ED85
```

However, both servers retain the same message header:

```text
Message-ID: <20261019135009.E25FAA2C41@mail-a.a.lab.test>
```

**Question:** When does `mail-a` remove the message from its queue?

Look for `status=sent` and the successful SMTP response from `mail-b`.

A successful transfer to the next SMTP server does **not necessarily mean final delivery to the recipient's mailbox**. In this experiment, final delivery is confirmed separately by the `local(8)` delivery log on `mail-b`.

## Experiment B: Destination server unavailable

Stop the receiving server **before** sending another message:

```bash
docker compose stop mail-b

printf 'From: tester@a.lab.test\nTo: student@b.lab.test\nSubject: Destination offline\n\nThis message should wait.\n' \
  | docker compose exec -T mail-a sendmail -i \
      -f tester@a.lab.test student@b.lab.test
```

Inspect the sender's queue and logs:

```bash
docker compose exec mail-a postqueue -p
docker compose logs --tail=80 mail-a
```

### Expected behavior

Because `mail-b` is unavailable, `mail-a` cannot complete the SMTP delivery.

After an unsuccessful delivery attempt, Postfix keeps the message in its queue and defers delivery.

Depending on the Docker network state, the failure may appear as a DNS lookup problem or a connection error.

If the queue initially shows a different state, wait briefly and inspect the logs again.

Restore the receiving server:

```bash
docker compose start mail-b
```

Request another delivery attempt:

```bash
docker compose exec mail-a postqueue -f
```

Verify the result:

```bash
docker compose logs --tail=80 mail-a mail-b

docker compose exec mail-b \
  sh -c 'tail -n 40 /var/mail/student'
```

`postqueue -f` requests a queue flush. It does not modify the Postfix configuration or guarantee delivery if the underlying problem remains.

**Do not reproduce these failure experiments on a production mail server.**

## Experiment C: Unknown recipient

This time, send a message to an address that does not correspond to a local user on `mail-b`:

```bash
printf 'From: tester@a.lab.test\nTo: unknown@b.lab.test\nSubject: Unknown recipient\n\nTesting permanent rejection.\n' \
  | docker compose exec -T mail-a sendmail -i \
      -f tester@a.lab.test unknown@b.lab.test
```

Inspect the logs:

```bash
docker compose logs --tail=80 mail-a mail-b
docker compose exec mail-a postqueue -p
```

With the default local recipient validation enabled, `mail-b` should reject the unknown recipient with a permanent SMTP error, typically `550 5.1.1`.

This illustrates the difference between:

- A **temporary failure** (`4xx`), where delivery may be retried.
- A **permanent rejection** (`5xx`), where retrying the same delivery without fixing the cause is generally not appropriate.

The original message is submitted locally to `mail-a`, so the initial `sendmail` command can succeed even though remote delivery later fails.

Postfix may generate a delivery status notification (bounce). Such a notification can itself be queued, so inspect the logs and queue instead of assuming that it has reached the sender's mailbox.

## Experiment D: One Message, Two Recipients, and a Bounce

So far, we have sent messages to a single recipient. Let's see what happens when **one message has two recipients**, but one of them does not exist.

We will also examine how Postfix generates a delivery failure notification (*bounce*) and what happens when that notification cannot be delivered either.

The first part of this experiment can be reproduced with the current lab configuration. The remaining steps document a NIS-related configuration problem encountered during development.

### Step 1: Send a message to two recipients

Make sure `mail-b` is running:

```bash
docker compose start mail-b
```

Submit one message with two envelope recipients:

```bash
printf 'From: tester@a.lab.test\nTo: student@b.lab.test, unknown@b.lab.test\nSubject: One message two recipients\n\nTesting multiple recipients.\n' \
  | docker compose exec -T mail-a sendmail -i \
      -f tester@a.lab.test \
      student@b.lab.test unknown@b.lab.test
```

Inspect the logs:

```bash
docker compose logs --tail=80 mail-a mail-b
```

### Step 2: One Queue ID, Two Delivery Results

In our experiment, `mail-a` accepted a single message with two recipients:

```text
78A70A2C4A: from=<tester@a.lab.test>, size=354, nrcpt=2 (queue active)
```

Postfix then recorded two different delivery results:

```text
78A70A2C4A: to=<unknown@b.lab.test>,
dsn=5.1.1, status=bounced

78A70A2C4A: to=<student@b.lab.test>,
dsn=2.0.0, status=sent
```

The first recipient does not exist. `mail-b` rejected the address with:

```text
550 5.1.1 Recipient address rejected:
User unknown in local recipient table
```

The rejection occurred during the SMTP `RCPT TO` command, before the message content was transferred.

The second recipient was accepted. `mail-b` assigned its own Queue ID:

```text
824A28ED83: from=<tester@a.lab.test>,
size=535, nrcpt=1 (queue active)
```

It then delivered the message to the local mailbox:

```text
824A28ED83: to=<student@b.lab.test>,
relay=local, dsn=2.0.0,
status=sent (delivered to mailbox)
```

**Conclusion:** A Queue ID on `mail-a` identifies a message that may have multiple recipients. Postfix tracks the delivery result for each recipient separately.

### Step 3: Postfix Generates a Bounce

Because one recipient was permanently rejected, `mail-a` generated a delivery status notification (DSN):

```text
78A70A2C4A: sender non-delivery notification: 84D34A2C52
```

This created a new message with its own Queue ID:

```text
84D34A2C52: from=<>, size=2461, nrcpt=1 (queue active)
```

Notice `from=<>`.

**Delivery status notifications use an empty envelope sender.** This prevents an undeliverable bounce from generating another bounce, which could otherwise lead to an endless loop of delivery failure notifications.

### Step 4: A Bounce Can Also Enter the Deferred Queue

During development, `mail-a` initially used the default alias lookup configuration:

```ini
alias_maps = hash:/etc/aliases, nis:mail.aliases
```

However, the container did not have a working NIS service.

When Postfix attempted to deliver the DSN locally, it reported:

```text
84D34A2C52: to=<tester@a.lab.test>,
relay=local, dsn=4.3.0,
status=deferred (alias database unavailable)
```

The original message had already been processed and removed from the queue.

The message remaining in `deferred` was **a new message containing the delivery failure notification**, not the original email.

This distinction is important when analyzing `mailq` output.

### Step 5: Fixing the Configuration and Retrying Delivery

We corrected the alias lookup configuration in `config/mail-a/main.cf`:

```ini
alias_maps = hash:/etc/aliases
```

Then rebuilt the sending container:

```bash
docker compose up -d --build --no-deps mail-a
```

We verified the effective configuration:

```bash
docker compose exec mail-a postconf alias_maps
```

Result:

```text
alias_maps = hash:/etc/aliases
```

The queue contents survived the rebuild because `/var/spool/postfix` is stored in a named Docker volume.

After Postfix restarted and another delivery attempt occurred, we inspected the logs. A queue flush can be requested with:

```bash
docker compose exec mail-a postqueue -f
docker compose logs --tail=40 mail-a
```

The logs showed:

```text
84D34A2C52: to=<tester@a.lab.test>,
relay=local, dsn=5.1.1,
status=bounced (unknown user: "tester")

84D34A2C52: removed
```

The alias database problem was resolved, but another issue became visible: the local user `tester` did not exist on `mail-a`.

Since the undeliverable message was itself a DSN with an empty envelope sender, Postfix did not generate another bounce.

### Conclusions

This experiment demonstrates four important behaviors:

1. A single queued message can have multiple recipients.
2. Different recipients can have different delivery results.
3. A delivery failure notification is a separate message with its own Queue ID.
4. A DSN can also enter the `deferred` queue, but failure to deliver a DSN does not generate another bounce.

**Important:** This experiment demonstrates partial success and a permanent recipient rejection. It does not yet demonstrate a case where one recipient has received the message while another remains in `deferred`. That requires a separate test with a controlled temporary delivery failure.

The current lab configuration already includes the corrected `alias_maps` setting. Therefore, users following these instructions will not encounter the historical NIS error unless they deliberately restore the old configuration. Reproducing that error is not required for the main experiment.

## Observed queue failures

During development, we deliberately triggered several delivery failures.

| Scenario | Observed DSN | Delivery status |
|---|---|---|
| DNS lookup failure for `mail-b` | `4.4.3` | `deferred` |
| Temporary receiver failure (`451`) | `4.3.0` | `deferred` |
| TCP connection refused on port 25 | `4.4.1` | `deferred` |
| Receiver restored; message accepted | `2.0.0` | `sent` |

These are observations from separate experiments. Not every failure mode can be reproduced simply by stopping the receiver container.

### How should we interpret these results?

- `4.4.3` — a directory system or DNS-related failure.
- `4.3.0` — a temporary mail system problem.
- `4.4.1` — a problem establishing the network connection.
- `2.0.0` — successful completion.

**The important lesson: `deferred` is a delivery state, not a diagnosis.**

Different failures can result in the same `deferred` state.

To determine the cause, inspect the diagnostic message and DSN recorded by Postfix.

Also remember that a DSN in a Postfix log is not always a code received directly from a remote SMTP server. Postfix can generate its own diagnostic status when, for example, the TCP connection cannot be established.

## Key concepts

| Component | Purpose |
|---|---|
| `smtpd` | Accepts incoming SMTP connections. |
| `pickup` | Collects locally submitted messages. |
| `cleanup` | Processes messages before queueing. |
| `qmgr` | Manages queued messages and delivery scheduling. |
| `smtp(8)` | SMTP client responsible for sending to the next server. |
| `local(8)` | Delivers messages to local mailboxes. |
| `relayhost` | Specifies an explicit next-hop SMTP server. |
| `deferred` | Indicates that delivery has been postponed. |
| Queue ID | Identifies a queued message within a Postfix instance. |
| Message-ID | Message header used to identify a message across systems; it is not necessarily unique or trustworthy. |

### When does responsibility transfer?

After the receiving SMTP server successfully accepts the message and returns its final `250` response following `DATA`, it assumes responsibility for further handling.

The sender can then remove its queued copy.

If the receiver accepts the message but the connection fails before the sender receives the acknowledgement, the sender may retry.

This is one reason why duplicate email delivery is possible.

## Troubleshooting

Inspect the queues:

```bash
docker compose exec mail-a postqueue -p
docker compose exec mail-b postqueue -p
```

Inspect recent logs:

```bash
docker compose logs --tail=100 mail-a mail-b
```

Check the effective configuration:

```bash
docker compose exec mail-a postconf -n
docker compose exec mail-b postconf -n
```

Inspect a queued message if you know its queue ID:

```bash
docker compose exec mail-a postcat -q QUEUE_ID
```

Replace `QUEUE_ID` with an actual identifier from that server's queue.

When investigating a delivery failure, ask:

1. Did Postfix accept the message?
2. Was the message placed in the queue?
3. Did the SMTP client attempt delivery?
4. Did the remote server respond, or did the connection fail?
5. Was delivery postponed, permanently rejected, or successful?
6. Did the next server merely accept the message, or was it delivered to the final mailbox?

## Limitations and future lessons

This laboratory intentionally excludes several production mail features:

- No external DNS/MX-based routing. The next hop is explicitly configured as `[mail-b]:25`, although Docker's internal DNS resolves the container name.
- No Dovecot or LMTP.
- No TLS or SMTP authentication.
- No anti-spam processing or public SMTP service.
- No host network mode or published SMTP ports.

Do not expose these containers directly to the Internet.

Possible future lessons include temporary versus permanent failures, retry scheduling and backoff, local delivery failures, Dovecot/LMTP, DNS/MX, and TLS/SASL.

These are ideas for future exercises, not requirements for running this lab.

## Further reading

- [Postfix mail queues and delivery failures — Logos (Polish)](https://logos.net.pl/pl/blog/postfix-smtp-kolejka-bledy-dostarczania/)
- [Understanding SMTP reply codes and enhanced status codes — Logos (Polish)](https://logos.net.pl/pl/blog/kody-bledow-smtp-jak-czytac-i-diagnozowac/)
- [Postfix documentation](https://www.postfix.org/documentation.html)
- [RFC 5321 — Simple Mail Transfer Protocol](https://www.rfc-editor.org/rfc/rfc5321.html)
- [RFC 3463 — Enhanced Mail System Status Codes](https://www.rfc-editor.org/rfc/rfc3463.html)

## Stopping and cleaning up

Stop the containers and remove the lab network:

```bash
docker compose down
```

Named volumes are preserved.

To remove containers, networks, and named volumes:

```bash
docker compose down -v
```

**Warning:** `down -v` also deletes the lab's stored messages and mail queues.

Do not use it if you want to preserve your experiment results.
