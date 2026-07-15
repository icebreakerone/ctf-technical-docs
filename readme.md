# Icebreaker One Core Trust Framework Technical Documentation

This repository contains the sources for a [mkdocs-material](https://squidfunk.github.io/mkdocs-material/)
build of the technical documentation for schemes within the Icebreaker One (IB1) Core Trust Framework (CTF)

## Viewing the docs

This documentation is hosted at [https://docs.core.trust.ib1.org](https://docs.core.trust.ib1.org)

## Installation

Building the documentation requires [Python 3](https://www.python.org/) or later.

### Install Python dependencies using pipenv

You will need to have [pipenv installed](https://pipenv.pypa.io/en/latest/installation.html)

```bash
pipenv install
```

## Local editing and testing

### Activate the pipenv shell (only required once)

```bash
pipenv shell
```

### Building a local version of the docs

Trust Framework documentation is versioned in step with the Registry for the Trust Framework. The documentation uses [mike](https://github.com/jimporter/mike) to manage versioning. This includes a local webserver you can use to test your changes.

```
mike deploy <version> [latest -u]
```

e.g. if the current edit is to version 2026-06-08 of the documents (corresponding to version 2026-06-08 of the Trust Framework Registry), which is also the latest (default served on the site):

```
mike deploy 2026-06-08 latest -u
```

The `-u` flag signals that the `latest` alias should be updated to the new deployment


`mike` automatically pushes updates to the `gh-pages` branch locally.

### Serving the site locally

```
mike serve
```

By default, this serves the docs on `http://localhost:8000`, but you can change this with `-a`/`--dev-addr`.

### Edit and test

Unlike `mkdocs`, `mike serve` doesn't automatically refresh when you make changes, so you need to quit `mike serve`,
run `mike deploy` then run `mike serve` again to view changes.

## Pushing completed updates

Each call to `mike deploy` adds a commit to the `gh-pages` branch. This leads to a lot of "junk commits" on the `gh-pages` branch
as you make and test edits. You should reduce these down to a single commit for each substantial change. To do this you will need to find the commit ID of the last commit before you started editing:

```
git checkout gh-pages
git log
```

This outputs a log of all commits on the branch in reverse order e.g.

```
commit xxxxxxxxxxxxxxxxxxxx4 (HEAD -> gh-pages)
Author: Alice <alice@example.com>
Date:   Fri Dec 16 17:09:23 2022 +0000

    Deployed xxxxxxx to 0.1 with MkDocs 1.4.2 and mike 1.1.2

commit xxxxxxxxxxxxxxxxxxxx3 (HEAD -> gh-pages)
Author: Alice <alice@example.com>
Date:   Fri Dec 16 17:09:22 2022 +0000

    Deployed xxxxxxx to 0.1 with MkDocs 1.4.2 and mike 1.1.2

commit xxxxxxxxxxxxxxxxxxxx2 (HEAD -> gh-pages)
Author: Alice <alice@example.com>
Date:   Fri Dec 16 17:09:21 2022 +0000

    Deployed xxxxxxx to 0.1 with MkDocs 1.4.2 and mike 1.1.2

commit xxxxxxxxxxxxxxxxxxxx1
Author: Bob <bob@example.com>
Date:   Thu Dec 15 14:12:36 2022 +0000

    Update the docs to something interesting

...
```

You need to rebase all your recent local changes on the last meaningful commit - "Update the docs to something interesting" in the above example -
and then re-commit and push that to `gh-pages` on the remote.

```
git reset --soft xxxxxxxxxxxxxxxxxxxx1
git commit -m "My meaningful commit"
git push
```

Finally, assuming you were working in `master`, check out `master` again so you don't accidentally edit the `gh-pages` branch.

```
git checkout master
```

### Deactivating the virtual environment

When you've finished editing, you can deactivate the virtual environment, or just close the Terminal window

```
deactivate
```
