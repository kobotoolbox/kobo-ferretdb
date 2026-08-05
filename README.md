# KoboToolbox on FerretDB

A minimal, single-host [KoboToolbox](https://www.kobotoolbox.org) deployment that
uses [FerretDB](https://github.com/FerretDB/FerretDB) in place of MongoDB.

It works, for a narrow definition of "works". You can create a form, deploy it,
submit data to it, see the submission in the web interface, and export it to XLSX,
with FerretDB storing the submissions in PostgreSQL and no MongoDB anywhere in the
stack. Nothing beyond that is tested, and FerretDB's own website has been expired
since May 2026 — see [Caveats](#caveats).

This is a **demonstration of compatibility, not a production deployment**. Every
password in this repository is an English word. See
[Not for production](#not-for-production).

## Why

MongoDB Community Server moved from the AGPL v3 to the
[Server Side Public License](https://github.com/mongodb/mongo/blob/master/LICENSE-Community.txt)
on 16 October 2018, and remains SSPL v1 today. The Open Source Initiative
[says the SSPL is not an open source license](https://opensource.org/blog/the-sspl-is-not-an-open-source-license),
and Debian [ruled it unsuitable for `main`](https://bugs.debian.org/915537) and
removed MongoDB from the archive in 2020. KoboToolbox stores every form submission
in MongoDB, so that license sits in the middle of an otherwise redistributable
stack.

FerretDB (Apache 2.0) speaks the MongoDB wire protocol and stores the data in
PostgreSQL using the [DocumentDB](https://github.com/documentdb/documentdb)
extension (MIT, originally from Microsoft, now a Linux Foundation project). If it
is a sufficient substitute, the MongoDB dependency can be removed without touching
a line of KoboToolbox application code — which is exactly what this repository
tests.

Read [Caveats](#caveats) before you get excited.

## What changed

This is a fork of [kobo-docker](https://github.com/kobotoolbox/kobo-docker) at tag
[`2.026.27e`](https://github.com/kobotoolbox/kobo-docker/releases/tag/2.026.27e).
The FerretDB change is deliberately isolated in a single commit, so that the
compatibility claim is easy to check — two files, 46 insertions, 20 deletions:

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

To confirm the data really is in PostgreSQL rather than MongoDB, read your
submission back out of it with `psql` after you have made one:

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
- Create a project and add a question.
- Deploy it. KPI reaches Enketo Express and gets back a working form URL
  (`http://ee.kobo.local/ee4gZfMO`), which loads and renders the form.
- Submit data, both from the Enketo web form and by `POST`ing to the OpenRosa
  `/submission` endpoint (HTTP 201, `Successful submission.`).
- The submission is visible through the KPI API and in the data table.
- Export to XLSX. The Celery task completes and the downloaded workbook contains
  the submitted value and the question label.

The submission was also read back out of the FerretDB side directly, both with
`mongosh` over the wire protocol and with `psql` against the DocumentDB tables, to
confirm it is genuinely stored there and not cached somewhere in Django.

## Caveats

**Only the above is tested.** This exercise deliberately covers the smallest
interesting path — one form, one question, one submission, one export. Large parts
of KoboToolbox touch MongoDB in ways this does not exercise at all, including
editing and deleting submissions, validation statuses, bulk operations, attachments
and media, the `/reports` aggregations, and REST services. Some of those use
aggregation pipeline stages and query operators that FerretDB may implement
partially or not at all. **Assume nothing beyond the list above works until you
test it.**

**No performance claims.** A single submission says nothing about how FerretDB
behaves under a real workload, and every query here ends up as SQL against
PostgreSQL. KoboToolbox's MongoDB indexes and query patterns were tuned against
MongoDB.

**No migration path.** This starts from an empty database. Moving an existing
KoboToolbox instance's submissions from MongoDB into FerretDB is a separate
problem and is not addressed here.

**Transactions are not supported.** FerretDB's own
[compatibility table](https://docs.ferretdb.io/migration/compatibility/) marks
`commitTransaction` and `abortTransaction` as not implemented, though sessions
themselves work. `bulkWrite` is also unimplemented, as are capped collections.
Change streams have been [an open issue since 2021](https://github.com/FerretDB/FerretDB/issues/175).
TTL indexes *are*
[supported](https://docs.ferretdb.io/guides/ttl-indexes/), with caveats: single
field only, `Date` type only, swept every 60 seconds, and documents inserted before
the index exists may not be affected.

Note that FerretDB publishes no stage-by-stage aggregation pipeline compatibility
matrix — its docs list nine stages "neutrally" and hedge with "some of the
aggregation stages". `$lookup`, `$facet`, `$merge` and friends are simply
unmentioned, as are `$where` and JavaScript execution. So there is no way to check
in advance whether a given KoboToolbox query will work; you have to run it. Since
FerretDB translates to SQL and embeds no JavaScript engine, anything relying on
server-side JS almost certainly will not.

**FerretDB's project health is worth a look before you depend on it.** As of
August 2026: the latest release is v2.7.0 from November 2025, `main` has had no
commits since February 2026, all four CI badges in the README are failing, and the
official website at ferretdb.com has been serving an expired-Squarespace 404 since
roughly May 2026 — [reported in May and still unanswered](https://github.com/FerretDB/FerretDB/issues/5650).
The [documentation](https://docs.ferretdb.io/) and blog subdomains are still up,
and the docs are versioned ahead of any shipped release. There is no announcement
of a shutdown or acquisition either way, so draw your own conclusions; the code is
Apache 2.0 and the DocumentDB extension underneath it is actively developed under
the Linux Foundation.

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
