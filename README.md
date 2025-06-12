# talk python - pytest pour faciliter les comptes rendu d outils métiers

# Dev

```bash
uv tool run mkslides serve --config-file src/mkslides_config.yml --strict src/slides.md --open
```

# Build

```bash
mkdir -p public

uv tool run mkslides build --config-file src/mkslides_config.yml --site-dir public --strict src/slides.md
```
