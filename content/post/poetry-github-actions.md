---
title: "Using Poetry in GitHub Actions (the easy way)"
date: 2021-11-02
keywords:
- python
- poetry
- GitHub actions
---

I've started using [poetry](https://python-poetry.org/) on a new Python project and wanted to integrate it within my GitHub Actions workflows
(which used to depend on pip).

I wanted to keep this simple and refrain from using [external actions](https://github.com/marketplace/actions/python-poetry-action). My requirements 
were simple:

- Install dependencies via poetry.
- Cache the virtualenv that poetry generates (this will speed up future runs).
- Run tests via `pytest`.

Here is what I came up with:
```
steps:
  - uses: actions/checkout@v1

  - name: Setup Python
    uses: actions/setup-python@v2
    with:
      python-version: 3.10.0

  - name: Install poetry
    run: |
      python -m pip install poetry==1.1.11

  - name: Configure poetry
    run: |
      python -m poetry config virtualenvs.in-project true

  - name: Cache the virtualenv
    uses: actions/cache@v2
    with:
      path: ./.venv
      key: ${{ runner.os }}-venv-${{ hashFiles('**/poetry.lock') }}

  - name: Install dependencies
    run: |
      python -m poetry install

  - name: Run tests
    run: |
      python -m poetry run python -m pytest -sxv

```

Turns out that installing poetry directly from pip and then using it as python module simplifies this whole thing. No need for external actions. 

Also, by running `poetry config virtualenvs.in-project true`, the virtualenv stays within the project root (defaults to `./venv`), which in 
turn makes it much easier to figure out the caching path. Anyway, I don't particularly like having my virtualenv outside my project root... I develop
on macos, run workflows on ubuntu and deploy to CentOS machines - who knows where these virtualevs could end up in?!
