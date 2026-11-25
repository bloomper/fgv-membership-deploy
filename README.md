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
