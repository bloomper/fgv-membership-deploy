# fgv-membership-deploy

Docker Compose for Medlemskap: `membership-api`, `membership-web` and `membership-mysql`, next to the other FGV
applications on the shared network `fgv-<DEPLOY_ENV>`. The product spec is in
[`fgv-membership-spec`](../fgv-membership-spec/docs/product-spec.md).

```
browsers ──▶ membership-web ──▶ membership-api ──▶ membership-mysql
other applications ──▶ membership-api (its own hostname, API_HOST)
membership-api ──▶ keycloak-server, spexregister-api                          (fgv network)
spexregister-api, revival-api ──▶ membership-api                              (fgv network)
```

The web reaches the API over the shared network. The API is also exposed on its own hostname, `API_HOST`, for the
applications outside FGV's network that call it: ones that take payments (`/api/external/payments` and
`/api/external/lookup`, see *Other applications that take payments*), and anything else that has a Keycloak token. The API
enforces authentication per endpoint: everything needs a Keycloak token except registration (guarded by Turnstile and rate
limits) and personal payment links (the token in the link is the key). `API_HOST` never has Cloudflare Access in front of
it: the applications that call it cannot complete a login.

## Before you start

- `fgv-infra-deploy` is running with `MEMBERSHIP_WEB_HOST` in its `.env`, and its `bin/init-secrets.sh` and
  `bin/configure-keycloak.sh` have run: they make the clients `fgv-membership` and `fgv-membership-service` and their
  secrets, and tell Keycloak to link *Register* to `MEMBERSHIP_URL/register`.
- A Cloudflare Turnstile widget for the web's hostname: its site key goes in `.env`, its secret key in
  `secrets/TURNSTILE_SECRET_KEY`. **Without the secret key the API refuses every registration.**
- `API_HOST` in `.env`: the hostname the API is reached on from outside (see *Staging* and *Production* for routing it).
- A mail relay (the one Keycloak uses). Medlemskap sends the verification, approval and rejection emails itself.

## Setting up

```shell
cp staging.env.example .env        # or production.env.example
nano .env
bin/init-secrets.sh                # copies the client secrets, generates the rest
nano secrets/TURNSTILE_SECRET_KEY
nano secrets/cloudflared.env       # staging only: TUNNEL_TOKEN=<token>
docker compose up -d
```

`bin/init-secrets.sh` copies the secrets of `fgv-membership` and `fgv-membership-service` from
`fgv-infra-deploy/secrets/clients/`, next to this checkout or one level further up (set `INFRA_DEPLOY` for other
layouts).

## Moving registration from Keycloak

Registration used to be Keycloak's, with Turnstile and approval emails from `fgv-keycloak`'s providers. Their keys
and recipients move here:

| Was (Keycloak's environment)                              | Is now                                        |
|-----------------------------------------------------------|-----------------------------------------------|
| `KC_SPI_FORM_ACTION__FGV_TURNSTILE__SITE_KEY`             | `TURNSTILE_SITE_KEY` in `.env`                |
| `KC_SPI_FORM_ACTION__FGV_TURNSTILE__SECRET_KEY`           | `secrets/TURNSTILE_SECRET_KEY`                |
| `KC_SPI_EVENTS_LISTENER__FGV_ADMIN_NOTIFICATION__RECIPIENTS` | `REGISTRATION_NOTIFY_RECIPIENTS` in `.env` |

Users who registered in Keycloak and still wait in the group `pending-approval` are handled in Medlemskap's user
administration like any other.

## Paying

`SWISH_NUMBER`, `BANKGIRO` and `BANK_ACCOUNT` are what members see on the payment page: pay with Swish, to the bankgiro with
the reference as the message, or (if a bank account is set) by a direct transfer, which may not be used. Without a Swish
number the page shows the bankgiro, the account and the reference only. Swish never tells FGV that a payment was made: the member says they paid, and the
treasurer records the payment under *Payments* in the web. Fees are set by an administrator under *Fees*.

The API reads spexare from Spexregister (`SPEXREGISTER_GRAPHQL_URL`, default `http://spexregister-api:8080/api/graphql`)
with `fgv-membership-service`'s `MEMBERSHIP` role, so Spexregister's API must be on the `fgv` network. After each change
to what counts as a membership it calls `SPEXREGISTER_NOTIFY_URL`, so Spexregister's copy follows within seconds.

## Other applications that take payments

An application where members pay (a shop on FGV's website, say) tells Medlemskap that someone paid, and can ask whether an
address has an account and a membership. Each such application gets its own Keycloak service client with the role
`PAYMENT` on `fgv-membership`; the first is `fgv-external-payment-service` (client credentials; its secret is
`clients/fgv-external-payment-service` in `fgv-infra-deploy`'s secrets). It gets a token from Keycloak and calls the API on
`API_HOST`. Another such application gets its own client with the role `PAYMENT`, and its client id is added to
`API_AUTHORIZED_PARTIES` in `.env`:

```
# Tell that someone paid for a year. paymentReference is the application's own id for the payment (an order or
# transaction id): it is shown to the treasurer, appears in the bookkeeping export, and together with the application
# makes telling the same payment twice harmless.
curl -X POST https://API_HOST/api/external/payments \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"email":"greta@example.org","firstName":"Greta","lastName":"Garbo","year":2027,"amount":200,"paymentReference":"order-1234"}'

# Is there an account, and is the address a member? year is optional (this year by default)
curl "https://API_HOST/api/external/lookup?email=greta@example.org&year=2027" -H "Authorization: Bearer $TOKEN"
```

`kind` in the payment may be `SUPPORT` (a support membership); `lifetime: true` is a life membership. For the payment, `201`
means recorded, `200` that it was told before or that the person already was a member that year, `400` that the request is
invalid (the message says why) and `503` that Spexregister cannot be reached; retry on any other failure. A new person gets
a spexare in Spexregister and an account with the *Default access* an administrator has chosen (Medlemskap web, *Default
access*), and two emails: one saying the account exists, one from the login service with a link to choose a password. The
lookup answers `{ "account": {"exists", "enabled"}, "spexare", "member", "active": {"year", "kind", "lifetime", "status"} }`.
The administrators (`REGISTRATION_NOTIFY_RECIPIENTS`) are emailed about every payment, from an application or reported by a
member. The full description is in the server's API guide.

## The board

The board's pages are under *Members* in the web: the members of a year with filters, the trend over the years, renewal
reminders, a page for each member (waive a fee, ask to pay with a personal link, record a payment that had no request),
and Excel lists of the members and, for the treasurer, of the year's payments. A *personal link* in an email opens
`https://WEB_HOST/pay/<token>`, a page that needs no login and shows only that payment request. The token is long and
random, its hash is all that is stored, and it works until the end of the year. Personal links and the exports are
served by the API under `/api/personal` and `/api/exports`; the web calls them over the shared network.

## Revivals, applications and settings

Nyuppsättning pays the year's FGV fee for its participants with one request: it calls the GraphQL mutation
`createRevivalPaymentRequest` as `fgv-revival-service` (role `REVIVAL` on `fgv-membership`) and shows the reference to the
director, who pays FGV the total; the treasurer matches the payment as usual. Swish number, bank account, who is emailed
about payments and the reminder time are set by an administrator under *Settings* in the web; the environment's
`SWISH_NUMBER`, `BANK_ACCOUNT` and `REGISTRATION_NOTIFY_RECIPIENTS` are only what is used until they are. The
maintenance jobs and the audit log are under *Maintenance* and *Audit log*.

## Data and backups

The database (memberships, payments, bank transactions and the audit log, which are FGV's bookkeeping) is in the named
volume `mysql_data`. Copying the files of a running MySQL server doesn't give a consistent backup, so
`bin/backup-db.sh` runs `mysqldump --single-transaction` into `backups/<timestamp>/membership-mysql.sql.gz`, which is
consistent without stopping anything. Run it nightly from the host user's crontab (`crontab -e`):

```
40 2 * * * /home/<user>/fgv-membership-deploy/bin/backup-db.sh >> /home/<user>/fgv-membership-deploy/backups/backup.log 2>&1
```

`BACKUP_KEEP_DAYS` sets how many days of dumps to keep (default 14). Copy `backups/` off the host as well, e.g. with
`rclone` or `rsync`: a backup on the same disk doesn't survive losing the disk. To restore a dump:

```shell
gunzip -c backups/<timestamp>/membership-mysql.sql.gz | docker compose exec -T membership-mysql \
  sh -c 'MYSQL_PWD=$(cat /run/secrets/MYSQL_ROOT_PASSWORD) mysql -uroot "$MYSQL_DATABASE"'
```

Every service logs to the console only; read it with `docker compose logs -f <service>`. Each keeps at most 5 × 10 MB
(`x-logging` in `compose.yaml`), so logs can't fill the disk.

## Staging

`compose.tunnel.yaml` adds `cloudflared`. In the tunnel's public hostnames route `WEB_HOST` to `http://membership-web:3000`
and `API_HOST` (no path) to `http://membership-api:8080`, uncached. Restrict who can reach staging with Cloudflare Access
as `spexregister-deploy`'s README describes: an application for `WEB_HOST` with an *Allow* policy for the testers, and
none for `API_HOST`.

## Production

Production's reverse proxy routes `WEB_HOST` to `membership-web:3000` and `API_HOST` to `membership-api:8080`, uncached.
The management port (`9000`, health and metrics) is not published and stays on the shared network. Schedule
`bin/backup-db.sh` in cron and copy `backups/` to off-host storage (see *Data and backups*), and pin `API_IMAGE` and
`WEB_IMAGE` to release versions.

## License

Apache 2.0.
