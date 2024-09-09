# Docs

Docs are generated using [MkDocs-Material](https://squidfunk.github.io/mkdocs-material/).
This is controlled by the [mkdocs.yml](pysystemtrade/mkdocs.yml) file in the root directory.

## Deployment

Deployment should occur automatically on push to master with [action.yml](.github/workflows/build-docs.yml). This should be configured in the REPO settings as well.

## Development

To locally test mkdocs, the best option is to run on the serve command with:

```bash
mkdocs serve -a <0.0.0.0:unused_port>
```

## Requirmenets
- mkdocs-material
- mkdocs-jupyter