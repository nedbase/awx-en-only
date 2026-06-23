# awx-en-only

A thin fork/overlay on top of [ansible/awx](https://github.com/ansible/awx) that
builds and publishes production AWX container images with **all locales except
`en-us` removed**.

## Why

AWX picks its UI language based on `navigator.languages` (the browser's language
preference list). There is no in-app language selector (see
[#15334](https://github.com/ansible/awx/issues/15334)), and the locale cannot be
overridden via `Accept-Language` headers or JS property patching at runtime.

The Dutch (`nl`) locale shipped with AWX 24.6.1 has poor translation quality
that degrades the UX for Dutch-speaking users. This fork strips every non-`en-us`
locale at build time so the UI always renders in English.

## How it works

The GitHub Actions workflow in `.github/workflows/build.yml`:

1. Checks out the upstream AWX source at the tag pinned in `awx-version.txt`.
2. Deletes every locale directory except English from **two** locations
   (which use different names for English):
   - `awx/locale/*/` — Django back-end `.po` / `.mo` files (keep `en-us`)
   - `awx/ui/src/locales/*/` — LinguiJS front-end `.po` catalogs (keep `en`)
3. Rewrites the UI's `i18nLoader.js` `locales` map down to `en` so that any
   non-English browser falls back to English instead of trying to load a
   catalog that was deleted.
4. Renders AWX's Jinja2 `Dockerfile.j2` template into a real `Dockerfile`.
5. Builds and pushes the image to GHCR as
   `ghcr.io/<your-org>/awx:<awx-version>`.

## Upgrading to a new AWX release

Edit `awx-version.txt`, commit, and push. The workflow will build the new
version automatically on the next push to `main`, or you can trigger it manually
via **Actions → Build and Publish AWX (en-US only) → Run workflow**.

## Using the image

In your AWX Operator `AWX` custom resource, set:

```yaml
spec:
  image: ghcr.io/<your-org>/awx
  image_version: 24.6.1
```

## Applying extra patches

Drop any `.patch` files (generated with `git format-patch` or `git diff`) into
the `patches/` directory. They will be applied to the upstream source before the
build. This is useful for cherry-picking upstream bug fixes or backporting your
own changes alongside the locale strip.

## Locale locations inside AWX (reference)

| Location | Format | Purpose |
|---|---|---|
| `awx/locale/<lang>/LC_MESSAGES/django.po` | GNU gettext | Back-end strings rendered server-side (English = `en-us`) |
| `awx/ui/src/locales/<lang>/messages.po` | LinguiJS | Front-end strings compiled into JS chunks (English = `en`) |

Both are compiled during the UI build (`compilemessages.py` + `lingui compile`)
inside the Docker image. Removing the directories is necessary but **not**
sufficient: the UI's `i18nLoader.js` `locales` map must also be trimmed to `en`,
otherwise a non-English browser tries to dynamically import a deleted catalog.
The workflow handles this automatically.

## Supported locales in AWX 24.6.1 (for reference)

- Back-end (`awx/locale/`): `en-us`, `es`, `fr`, `ja`, `ko`, `nl`, `zh`
- Front-end (`awx/ui/src/locales/`): `en`, `es`, `fr`, `ja`, `ko`, `nl`, `zh`,
  plus `zu` (a pseudo-locale used only for translation testing)
