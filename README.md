# labs.inqk.net

This is the website at labs.inqk.net. It hosts experiments. It is intended to be
updated using [Tenter][rg].

[rg]: https://rubygems.org/gems/tenter/

## Passwords

Each experiment is shut behind its own cookie, checked by nginx. There is no
script involved: nginx cannot receive an HTML form without something behind it,
so the login is a password prompt on a single unlock URL, and the answer to it
is a cookie that lasts a year.

To protect a new experiment, for each one:

1. Generate a secret and a password file on the server:

        $ openssl rand -hex 32
        $ mkdir -p /var/www/labs.inqk.net/auth
        $ htpasswd -c /var/www/labs.inqk.net/auth/<name> <user>

2. Copy the `map` block and the two `location` blocks for `foobar` in
   `nginx.conf.example`, replacing the name and pasting the secret into both
   the map and the `Set-Cookie`.

Then visit `https://labs.inqk.net/<name>/unlock` once. Every page under
`/<name>/` loads without a prompt from then on, on that browser.

Two things this depends on:

- **HTTPS.** The password and the cookie both travel in clear over plain HTTP,
  and the `Secure` flag stops the cookie being sent at all without TLS.
- **The `^~` on the directory's location.** A regex location is matched ahead
  of a prefix one, so without `^~` the `\.(js|css|png|...)` rule would catch
  every stylesheet and image inside a protected directory and serve it to
  anyone.

There is no logout. Changing the secret in the config invalidates every cookie
issued from it.

## Adding an experiment

There is no page at the root: <https://labs.inqk.net/> returns 404 and so does
any directory without an `index.html`. Each experiment is self-contained, lives
in its own directory under `src/` and is served at
`https://labs.inqk.net/<name>/`:

    src/
    └─my-experiment/
      ├─index.html
      └─...

Nothing needs to be registered anywhere. Creating the directory is enough.

## Deployment

The site is deployed to `/var/www/labs.inqk.net` on the server. The `public`
symlink points at `src`, which is what nginx serves (see
`nginx.conf.example`).

Pushing to `origin` triggers a GitHub webhook to
<https://hooks.inqk.net/run/update/in/labs.inqk.net/>, which runs
`commands/update` on the server. That command is a `git pull`, so no build step
is required. The webhook's secret is the value of `secret` in `hooks.yaml`,
which is not tracked by git; use `hooks.yaml.example` as the template.
