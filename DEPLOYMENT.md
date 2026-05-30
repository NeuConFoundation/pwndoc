# PwnDoc Fork — GMF Red Cloud Deployment Target

This is a forked + GMF-customized version of [PwnDoc](https://github.com/pwndoc/pwndoc), the open-source pentest report management tool. The fork lives here at the top level (outside `redteam-infrastructure/`) because:

1. **Upstream-merge hygiene** — keeping it separate from the infra mono-repo means we can cleanly rebase from upstream without entangling Bicep/PowerShell/Terraform-style infra code.
2. **License separation** — PwnDoc is GPL; our infra docs/IaC are GMF-internal.
3. **Repo discipline** — when this hits ADO, this becomes its own repo inside the `RedCloud` ADO project, alongside the main `redteam-infrastructure` repo.

## Where this gets deployed

**Target:** GMF Red Cloud Zone 4, VM `RC-Z4-PWN-01` in `snet-pwndoc`.

**Deployment artifacts:**
- `redteam-infrastructure/enterprise-red/red-cloud/hosted-services/pwndoc/pwndoc-design.md` — full design doc (Zone 4 placement rationale, two-access-paths, NSG rules, cert via Let's Encrypt + Cloudflare DNS-01, KV-stored secrets)
- `redteam-infrastructure/enterprise-red/red-cloud/hosted-services/pwndoc/deploy.sh` — bash deployment script run on the VM at provisioning
- `redteam-infrastructure/enterprise-red/red-cloud/hosted-services/pwndoc/docker-compose.prod.yml` — production compose file (host nginx → backend on 127.0.0.1:8080, MongoDB)

## Build pipeline (planned — locked 2026-05-29)

A separate ADO pipeline `pwndoc-image-build.yml` will live in this repo once it's in ADO. It:

1. Builds Docker images for backend + frontend from this source
2. Scans for vulnerabilities (govulncheck-equivalent for Node, `npm audit --audit-level=high`)
3. Tags with semver + git SHA
4. Pushes to **Azure Container Registry (`acr-redcloud.azurecr.io`)** in the Red Cloud subscription
5. Updates a manifest file that the Red Cloud `deploy.sh` reads to pull the blessed version

See `redteam-infrastructure/enterprise-red/red-cloud/deployment/pipelines/pwndoc-image-build.yml` (stub) for the pipeline definition.

## Identity + access

- Custom rebranding / GMF UI changes to upstream PwnDoc happen here
- Operator + corp-user auth flow → see `red-cloud/deployment/identity/identity.md` §6.2
- Corp users (GRC, auditors, blue team) reach PwnDoc via hub peering through Palo Alto NVA inspection — they do NOT come through this source repo, they hit the deployed instance.
- SAML/SSO to Entra is a future enhancement (R-09 in `red-cloud/open-items.md`).

## Upstream sync

When upstream PwnDoc releases a new version:
1. `git fetch upstream` (where upstream is the original PwnDoc repo)
2. Rebase customizations on the new upstream tag
3. Test in a non-production Red Cloud sub or local docker-compose
4. Rebuild images via `pwndoc-image-build.yml` pipeline
5. Deploy to RC-Z4-PWN-01 via `pwndoc-deploy.yml` pipeline (zero-downtime if possible)

## Related decisions

| Open item | Status | Notes |
|-----------|--------|-------|
| D-7 (ADO project name) | RESOLVED 2026-05-29 | `RedCloud` project hosts this fork's repo + the main mono-repo |
| D-33 (corp source CIDRs for PwnDoc access) | Pending GMF Network team | Required before corp users can hit RC-Z4-PWN-01 |
| D-34 (Let's Encrypt + Cloudflare DNS-01 cert) | RESOLVED 2026-05-26 | Cert pipeline documented in `pwndoc-design.md` §5.1 |
| R-09 (SAML/SSO to Entra) | Open | Future enhancement, PwnDoc v1.x SAML support pending |
