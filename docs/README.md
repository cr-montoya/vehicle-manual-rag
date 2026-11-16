# Diagrams

| Source (editable) | Export used in the README |
| --- | --- |
| [`arquitectura.drawio`](arquitectura.drawio) | [`arquitectura.png`](arquitectura.png): AWS architecture, with the question, ingestion and alert paths |
| [`flujo-rag.drawio`](flujo-rag.drawio) | [`flujo-rag.png`](flujo-rag.png): what happens at deploy, at ingestion and on each question |

Edit the `.drawio` files in [diagrams.net](https://app.diagrams.net/) or the draw.io desktop app. The architecture diagram uses the AWS `aws4` shapes bundled with draw.io.

## Regenerating diagrams

Run from the repository root. It only needs Docker; nothing is installed on the host:

```bash
docker run --rm -v "$PWD":/data rlespinasse/drawio-export:v4.60.0 \
  -f png -s 2 -b 20 --remove-page-suffix -o . docs
```

This writes one PNG next to each `.drawio` file in `docs/`, at 2x scale with a 20 px border and a white background, keeping the file names the README links to.

The image tag `v4.60.0` was published on 2026-09-14. Digest: `sha256:2c133c2bbb42ba97fb3c8aca83fd62be3927fef8c7caeba2673d4092b7f6c578`. See the [export image documentation](https://github.com/rlespinasse/drawio-export/blob/v4.x/README.adoc).

On Linux, the PNG files may end up owned by `root` if they did not exist before; run `sudo chown "$USER" docs/*.png` if needed.
