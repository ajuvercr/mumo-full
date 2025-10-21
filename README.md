# mumo-full

For best results first start everything without pipeline and solid.

```bash
docker compose up nginx db php adminer graphs
```

Then you can populate the database using the ingest scripts from `./dashboard/scripts`.

Finally create and publish the LDES.

```bash
docker compose up pipeline solid
```
