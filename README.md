# KoboToolbox on FerretDB

A minimal, single-host [KoboToolbox](https://www.kobotoolbox.org) deployment that
uses [FerretDB](https://github.com/FerretDB/FerretDB) in place of MongoDB.

No changes are made to the KoboToolbox application, i.e.
https://github.com/kobotoolbox/kpi. Only to the Docker deployment
configuration, https://github.com/kobotoolbox/kobo-docker, is modified.

You can create a form, deploy it, submit data to it, see the submission in the
web interface, and export it to XLSX, with FerretDB storing the submissions in
PostgreSQL and no MongoDB anywhere in the stack. Nothing beyond that has yet
been tested.

This is a **demonstration of compatibility, not a production deployment**. Every
password in this repository is an easily readable (and guessable) word. See
[Not for production](#not-for-production).

## Why

The Digital Public Goods Alliance deems MongoDB insufficiently "open-source"
and [suggests
FerretDB](https://github.com/DPGAlliance/dpg-resources/tree/5ae7fa39600d19adb3b2e2c8d31ac49aef5ad3a9/docs/platform-independence#examples-of-common-dependencies)
as an alternative. MongoDB Community Server moved from the AGPL v3 to the
[Server Side Public
License](https://github.com/mongodb/mongo/blob/master/LICENSE-Community.txt) on
16 October 2018, and remains SSPL v1 today. The Open Source Initiative [does
not recognize the
SSPL](https://opensource.org/blog/the-sspl-is-not-an-open-source-license).
KoboToolbox, via its predecessor Formhub from the Sustainable Engineering Lab
at Columbia University, has been using MongoDB [since January
2011](https://github.com/SEL-Columbia/formhub/commit/d4699ddc0).

FerretDB (Apache 2.0) strives to speak the MongoDB wire protocol and stores the
data in PostgreSQL using the
[DocumentDB](https://github.com/documentdb/documentdb) extension (MIT,
originally from Microsoft, now a Linux Foundation project).

## What changed

This is a fork of [kobo-docker](https://github.com/kobotoolbox/kobo-docker) at tag
[`2.026.27e`](https://github.com/kobotoolbox/kobo-docker/releases/tag/2.026.27e).
The FerretDB change is deliberately isolated in a single commit, so that the
compatibility claim is easy to check:

```console
$ git log --format='%H' --grep='^Replace MongoDB with FerretDB' | xargs git show
```

The substance of it is in [`docker-compose.backend.yml`](docker-compose.backend.yml):
the `mongo:8.0` service is replaced by `ghcr.io/ferretdb/ferretdb:2.7.0` plus a
`postgres_ferretdb` service running FerretDB's PostgreSQL+DocumentDB image. The
service keeps the name `mongo` and the `mongo.kobo.private` network alias, so KPI,
KoboCAT, Celery and `wait_for_mongo.bash` reach it without knowing the difference.
In [`env/envfiles/databases.txt`](env/envfiles/databases.txt), `MONGO_DB_URL`
points at it with an ordinary `mongodb://` URL.

**No KoboToolbox application code is modified, patched or rebuilt.** The images are
the published `kobotoolbox/kpi:2.026.27e` and
`kobotoolbox/enketo-express-extra-widgets:7.6.3` from Docker Hub.

The rest of the diff is unrelated to FerretDB — it is the work of making
kobo-docker run standalone on one host without
[kobo-install](https://github.com/kobotoolbox/kobo-install). See
[Differences from upstream kobo-docker](#differences-from-upstream-kobo-docker).

## Not for production

Everything sensitive in this repository is a well-known example value, chosen to be
readable rather than secret:

| Setting | Value |
| --- | --- |
| Superuser | `super` / `admin` |
| PostgreSQL (Django) | `kobo` / `hedgehog` |
| PostgreSQL (FerretDB) | `ferret` / `badger` |
| Redis | `squirrel` |
| Enketo API key | `enketorabbit` |
| `DJANGO_SECRET_KEY` | `insecure-demo-secret-key-not-for-production-use` |

`DJANGO_DEBUG` is off and `DJANGO_ALLOWED_HOSTS` is restricted, but the stack is
otherwise wide open: no TLS, no real secrets, no backups configured, and email
written to the container log instead of sent. **Do not expose it to a network you
do not control.**

## Quickstart

Requires Docker with Compose v2, on Linux, and about 8 GB of disk for the images.

Add the demo hostnames to your `/etc/hosts`:

```
127.0.0.1  kf.kobo.local kc.kobo.local ee.kobo.local
```

Then:

```console
$ git clone https://github.com/kobotoolbox/kobo-ferretdb.git
$ cd kobo-ferretdb
$ docker compose up -d
```

The `.env` file in the repository sets `COMPOSE_FILE` to both compose files, so
`docker compose up` needs no arguments. First boot takes a few minutes to pull the
images; after that the stack reaches HTTP 200 in about 80 seconds.

Open <http://kf.kobo.local> and log in:

| Username | Password |
| --- | --- |
| `super` | `admin` |

To confirm the data really is in FerretDB's PostgreSQL instance rather than
MongoDB, read your submission back out of it with `psql` after you have made
one:

```console
$ docker compose exec postgres_ferretdb psql -U ferret -d postgres -tAc \
    "select documentdb_core.bson_to_json_string(document)
       from documentdb_api.collection('formhub', 'instances')"
{ "_id" : { "$numberInt" : "1" }, "formhub/uuid" : "d66aa8a4…",
  "favorite_animal" : "ferret", … }
```

To stop:

```console
$ docker compose down
```

Data lives in bind mounts under `.vols/`, not in named Docker volumes, so
`docker compose down -v` does **not** discard it. To reset completely, remove that
directory — parts of it are owned by container users, so this needs root:

```console
$ docker compose down
$ sudo rm -rf .vols log backups
```

## Verified working

Tested from a clean slate — no data directories, no manual database setup — with a
single `docker compose up -d`:

- All 13 containers start and stay up.
- The `formhub` database and its `instances` collection **are created
  automatically** on first write. No manual `use formhub` or seed document is
  needed; pymongo 4.10.1 (as shipped in the KPI image) authenticates against
  FerretDB 2.7.0 with a plain `mongodb://` URL and no explicit `authMechanism`.
- Log in as the superuser; the React interface loads.
- Access the Django admin interface (`/admin`) and add a regular user.
- Log in as that regular user.
- Create a project and add a question.
- Deploy it. KPI reaches Enketo Express and gets back a working form URL
  (`http://ee.kobo.local/…`), which loads and renders the form.
- Submit data, both from the Enketo web form and by `POST`ing to the OpenRosa
  `/submission` endpoint (HTTP 201, `Successful submission.`).
- The submission is visible through the KPI API and in the data table.
- Export to XLSX. The Celery task completes and the downloaded workbook contains
  the submitted value and the question label.

The submission was also read back out of the FerretDB side directly, both with
`mongosh` over the wire protocol and with `psql` against the DocumentDB tables, to
confirm it is genuinely stored there and not cached somewhere in Django.

## Differences from upstream kobo-docker

Besides swapping MongoDB for FerretDB, this fork drops the parts of upstream that
exist to support multi-host production deployments, so that the demo is one clone
and one command.

**No kobo-install.** Upstream expects
[kobo-install](https://github.com/kobotoolbox/kobo-install) to generate a sibling
`kobo-env/` directory of environment files with random secrets. Here they are
committed in [`env/`](env/) with example values. `env/envfiles/domains.txt` is a new
file: every service reads it, but upstream ships no template for it because
kobo-install writes it.

**One Compose project instead of two.** Upstream runs
`docker-compose.backend.yml` and `docker-compose.frontend.yml` as separate projects
on separate hosts, joined by an externally created network. Here `.env` sets
`COMPOSE_FILE` to both and they share one bridge network, keeping the
`*.kobo.private` and `*.docker.internal` aliases upstream's configuration expects.

**Network aliases instead of `extra_hosts`.** Parts of KoboToolbox deliberately use
*public* URLs for server-to-server calls — KPI posts to `ENKETO_URL` to deploy a
form, and Enketo posts submissions back to `KOBOCAT_URL`. Upstream makes those
resolve inside containers by giving each one an `extra_hosts` entry pointing at the
host's LAN IP, autodetected by kobo-install. This fork instead aliases
`kf.kobo.local`, `kc.kobo.local` and `ee.kobo.local` onto the nginx container, so
Docker's own DNS answers.

That is what makes `127.0.0.1` usable in `/etc/hosts`: containers no longer consult
the host's name resolution at all, so the host is free to point those names at
loopback. **This is verified on Linux only.** Where the Docker daemon runs inside a
VM — Docker Desktop on macOS or Windows — `127.0.0.1` inside the VM is not the same
address as on your desktop, and you may need the host's LAN IP instead. That is the
limitation upstream's `extra_hosts` approach works around.

**None of upstream's `mongo/` scripts are used.** They configure a replica set and
create users through `mongosh`, which FerretDB neither needs nor supports; FerretDB
authenticates clients against PostgreSQL roles instead. The directory is left in
place, unreferenced, to keep the diff small.

**Nothing operational is configured.** No S3, no backup schedules, no TLS, no
maintenance mode, and email goes to the log. Upstream's machinery for all of that
is still in the repository but unused.

## Versions

| Component | Version |
| --- | --- |
| kobo-docker base | `2.026.27e` |
| KPI (Django 5.2, pymongo 4.10.1) | `kobotoolbox/kpi:2.026.27e` |
| Enketo Express | `kobotoolbox/enketo-express-extra-widgets:7.6.3` |
| FerretDB | `ghcr.io/ferretdb/ferretdb:2.7.0` |
| FerretDB storage | `ghcr.io/ferretdb/postgres-documentdb:17-0.107.0-ferretdb-2.7.0` |
| PostgreSQL (Django) | `postgis/postgis:14-3.2` |
| Redis | `redis:7.2` |
| nginx | `nginx:1.27` |

FerretDB 2.7.0 reports itself as MongoDB 7.0.77, wire protocol version 21.

Version matters here: **FerretDB 1.x will not work.** It supported only `PLAIN`
authentication, so pymongo's default SCRAM negotiation fails against it. FerretDB
2.x supports `SCRAM-SHA-256`, which pymongo negotiates automatically, which is why
`MONGO_DB_URL` needs no `authMechanism` parameter.

## License

Upstream kobo-docker's license applies; see [LICENSE](LICENSE).
