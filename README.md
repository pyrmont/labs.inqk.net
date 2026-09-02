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
        $ printf '<user>:%s\n' "$(openssl passwd -apr1)" > /var/www/labs.inqk.net/auth/<name>

2. Copy the `map` block and the three `location` blocks for `foobar` in
   `nginx.conf.example`, replacing the name and pasting the secret into both
   the map and the `Set-Cookie`, which lives in the named location rather
   than in the one that asks for the password.

Then visit `https://labs.inqk.net/<name>/unlock` once. Every page under
`/<name>/` loads without a prompt from then on, on that browser.

### Adding and removing users

An auth file is one `user:hash` line per person. Append to add someone to an
experiment that already has a file, or the `>` above will throw away everyone
already in it:

    $ printf '<user>:%s\n' "$(openssl passwd -apr1)" >> /var/www/labs.inqk.net/auth/<name>

nginx reads the file from the top and stops at the first line whose name
matches, so a second line for a name already present is never reached.
Changing a password means deleting the old line first:

    $ sed -i '/^<user>:/d' /var/www/labs.inqk.net/auth/<name>

Deleting the line alone removes a user. None of this needs a reload: the file
is read on each request that needs it, which is also why it wants to stay
small and world-readable. What a plain `>` leaves it as depends on the umask
-- 664 on the server as it stands -- and any of those are fine so long as the
world-read bit survives: nginx runs as `nginx`, and a file it cannot read is a
500 on the unlock URL rather than a prompt.

Removing a user shuts nobody out, though. The password is asked for only at
the unlock URL, so anyone already holding a cookie keeps it.

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
