# Community Application Catalog

An **opt-in** catalog of self-contained **application stacks** for the
[Insula](https://github.com/insulahq/insula) Kubernetes hosting platform — WordPress, Nextcloud,
Gitea, and friends. Each entry bundles its own components (app + database + cache) and is deployed
via a Helm chart.

This repository is **not** installed by default. The default
[`insulahq/application-catalog`](https://github.com/insulahq/application-catalog) ships only
platform primitives (runtimes, databases, services). Add this repo when you want ready-to-run apps.

## Enable it

In the admin panel: **Applications → Repositories → Add repository**, then paste:

```
https://github.com/insulahq/application-catalog-community
```

## What's here

`wordpress`, `nextcloud`, `jitsi`, `moodle-bitnami`, `coturn`, `prestashop`,
`vaultwarden`, `gitea`, `bookstack`, `n8n`, `uptime-kuma`, `discourse`, `plausible`, `wiki-js`,
`immich`, `rocketchat`, `directus`, `paperless-ngx`, `invoiceninja`.

## Images

These stacks reference **upstream, third-party** container images (e.g. `wordpress:…`,
`nextcloud:…`, `ghcr.io/immich-app/…`). They are **not** built or hardened by the platform — you
are trusting the upstream project. There is no image-build pipeline in this repo.

## Layout

```
<entry>/
  manifest.json   # catalog metadata (type: application, multi-component)
  chart/          # Helm chart
  icon.png
catalog.json      # index: ordered list of entry slugs
schema/           # manifest JSON schema (shared with the official catalog)
scripts/          # validate / format tooling
```

## Validate

```bash
npm ci
npm run format:check
npm run validate
```
