# Day 28 — Resources: Helm & Kustomize

## Primary (assigned)

- **Helm docs — Chart Development Guide** (helm.sh/docs/chart_template_guide) — free, the assigned starting point. Covers chart structure, the templating engine, values precedence, and hooks with the exact detail referenced throughout this day's notes.

## Deepen your understanding

- **Helm docs — Helm and OCI** (helm.sh/docs/topics/registries) — the authoritative reference for `helm push`/`pull`/`registry login` and OCI-based chart distribution covered in file 2.
- **Kustomize docs** (kubectl.docs.kubernetes.io/references/kustomize) — the full reference for bases, overlays, strategic merge vs JSON 6902 patches, and components, covered in file 3.
- **"Kustomize vs Helm" — comparisons from the Kubernetes SIG-CLI community** (search the kustomize GitHub repo's FAQ/docs) — written by the maintainers of both ecosystems, useful for a balanced, non-marketing view of tradeoffs.
- **helm-docs GitHub repo** (github.com/norwoodj/helm-docs) — README covers the comment-annotation syntax used to auto-generate chart documentation, referenced in file 2's best-practices section.

## Reference / lookup

- `helm show values <chart>` / `helm show chart <chart>` — inspect any chart's default values and metadata without fully downloading/installing it.
- **Artifact Hub** (artifacthub.io) — searchable index of public Helm charts (and OCI-based charts) across many registries; useful for finding a well-maintained third-party chart before writing your own.

## Practice

- **KillerCoda / KodeKloud free Helm and Kustomize labs** — browser-based scenarios for practicing chart templating and Kustomize overlays without local setup.
- Package a real project you've built (or the internship app referenced in this day's assigned activity) as a Helm chart with real dev/staging/prod values files, exactly as done in this day's lab — applying the concepts to something you already understand end-to-end is the fastest way to make templating/values precedence concrete.
