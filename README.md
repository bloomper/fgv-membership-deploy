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
