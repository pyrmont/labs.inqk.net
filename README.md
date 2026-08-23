# labs.inqk.net

This is the website at labs.inqk.net. It hosts experiments. It is intended to be
updated using [Tenter][rg].

[rg]: https://rubygems.org/gems/tenter/

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
