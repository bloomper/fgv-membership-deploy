# fgv-membership-deploy

Docker Compose for Medlemskap: `membership-api`, `membership-web` and `membership-mysql`, next to the other FGV
applications on the shared network `fgv-<DEPLOY_ENV>`. The product spec is in
[`fgv-membership-spec`](../fgv-membership-spec/docs/product-spec.md).

Only `membership-web` is reached from outside. The API stays on the shared network, where Keycloak, and later Spexregister
and Nyuppsättning, call it.

## Before you start

- `fgv-infra-deploy` is running with `MEMBERSHIP_WEB_HOST` in its `.env`, and its `bin/init-secrets.sh` and
  `bin/configure-keycloak.sh` have run: they make the clients `fgv-membership` and `fgv-membership-service` and their
  secrets, and tell Keycloak to link *Register* to `MEMBERSHIP_URL/register`.
- A Cloudflare Turnstile widget for the web's hostname: its site key goes in `.env`, its secret key in
  `secrets/TURNSTILE_SECRET_KEY`. **Without the secret key the API refuses every registration.**
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

## License

Apache 2.0.

## Paying

`SWISH_NUMBER` and `BANK_ACCOUNT` are what members see on the payment page; without a Swish number the page shows the
bank account and the reference only. Swish never tells FGV that a payment was made: the member says they paid, and the
treasurer records the payment under *Payments* in the web. Fees are set by an administrator under *Fees*.

The API reads spexare from Spexregister (`SPEXREGISTER_GRAPHQL_URL`, default `http://spexregister-api:8080/api/graphql`)
with `fgv-membership-service`'s `MEMBERSHIP` role, so Spexregister's API must be on the `fgv` network. After each change
to what counts as a membership it calls `SPEXREGISTER_NOTIFY_URL`, so Spexregister's copy follows within seconds.

## The WordPress shop

The shop tells the API that someone paid with `POST https://API_HOST/api/external-payments` and the access token of the
`fgv-wordpress-service` client (client credentials; its secret is `clients/fgv-wordpress-service` in
`fgv-infra-deploy`'s secrets). The API's address is the one the web uses; expose `/api/external-payments` the same way
as `/api/registration`.

```
curl -X POST https://API_HOST/api/external-payments \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"email":"greta@example.org","firstName":"Greta","lastName":"Garbo","year":2027,"amount":200,"reference":"order-1234"}'
```

`201` means recorded, `200` that it was told before or that the person already was a member that year, `400` that the
request is invalid (the message says why). Retry on any other answer: the same `reference` never records twice. A new
person gets an account with the *Default access* an administrator has chosen (Medlemskap web, *Default access*), and two
emails: one saying the account exists, one from the login service with a link to choose a password. The administrators
(`REGISTRATION_NOTIFY_RECIPIENTS`) are emailed about every payment, from the shop or reported by a member.

## The board

The board's pages are under *Members* in the web: the members of a year with filters, the trend over the years, renewal
reminders, a page for each member (waive a fee, ask to pay with a personal link, record a payment that had no request),
and Excel lists of the members and, for the treasurer, of the year's payments. A *personal link* in an email opens
`https://WEB_HOST/pay/<token>`, a page that needs no login and shows only that payment request. The token is long and
random, its hash is all that is stored, and it works until the end of the year. Personal links and the exports are
served by the API under `/api/personal` and `/api/exports`; only `/api/personal` is open without a token, so expose
that path (and the web) but nothing else of the API.

## Revivals, applications and settings

Nyuppsättning pays the year's FGV fee for its participants with one request: it calls the GraphQL mutation
`createRevivalPaymentRequest` as `fgv-revival-service` (role `REVIVAL` on `fgv-membership`) and shows the reference to the
director, who pays FGV the total; the treasurer matches the payment as usual. Swish number, bank account, who is emailed
about payments and the reminder time are set by an administrator under *Settings* in the web; the environment's
`SWISH_NUMBER`, `BANK_ACCOUNT` and `REGISTRATION_NOTIFY_RECIPIENTS` are only what is used until they are. The
maintenance jobs and the audit log are under *Maintenance* and *Audit log*.
