# deployments

What runs where. One Cloud Run service file per repo, service and environment: `apps/<repo>/<service>/<env>.yaml` (Knative format, applied with `gcloud run services replace`).

This repo is public on purpose: on GitHub's free plan only public repos get environment approvals, which gate prod. It holds no secret values, only secret names.

- Files are created by Terraform in the `platform` repo when an app is added. After that, this repo owns them.
- Images are tagged with the app's release version, e.g. `.../menu/web:v1.4.0`. An app's CI sets it in every `dev.yaml` when its release PR is merged. Dev deploys straight away.
- Prod only deploys from a manual run of `promote`. Pick the app and service (or `all`) from the dropdowns; the lists come from `platform/apps.yaml` (the workflow is generated there, don't edit it here). It copies the dev version to `prod.yaml` and deploys the chosen prod files, also when the version is unchanged. Prod runs wait for approval on the `prod` environment.
- Any other change (memory, scaling, env vars, the deploy switch) is a normal edit to the file. Merging it deploys dev files right away; prod files go out on the next `promote`.
- `metadata.labels.public: "true"` makes the service public (`allUsers` invoker). `"false"` makes it internal: only
  the identities for that file in the `SERVICE_INVOKERS` secret may call it, and it gets no domain mapping. Both
  labels, `platform-access` and the `platform:` block come from `apps.yaml` (`internal`, `callers`, `testers`);
  don't edit them here. The deploy sets the service's IAM policy from them every time.
- Off switch: label `deploy: "false"` here, or `platform-deploy: "false"` (written by the platform from `apps.yaml`; don't edit it here). Either one deletes the Cloud Run service and its domain mapping; removing the label deploys it again.
- If `BETTER_AUTH_URL` is a custom host (not `*.run.app`), deploy creates a Cloud Run domain mapping for it. The
  `deployer` service account must be an owner of the domain in Search Console, and the host needs a CNAME to
  `ghs.googlehosted.com.`

Rollback: set the image tag in `prod.yaml` back to an earlier `vX.Y.Z` and merge.

Removing a file does not delete the Cloud Run service. Delete it with `gcloud run services delete`.
