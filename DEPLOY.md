# Staging deployment runbook

## Trigger
Every push to `main` runs `test -> build -> deploy`. Pull requests only run `test`.
The `deploy` job starts only after the image is published, calls the Render deploy API,
waits until Render reports the deploy as `live`, then verifies the service.

## Target
- Render Web Service `pipeline-lab-staging`, created from the public GHCR image
  `ghcr.io/sagalou/holbertonschool-hbtn-devops-pipeline-lab:latest`.
- The package was made public deliberately: the image holds application code and
  production dependencies only, no credentials.
- GitHub secrets: `RENDER_API_KEY`, `RENDER_SERVICE_ID`.
- GitHub variable: `STAGING_URL` (no trailing slash).

## Database configuration
- Disposable Render PostgreSQL `pipeline-lab-db`, same region as the web service.
- The web service reads `DATABASE_URL`, set in the Render dashboard to the database's
  internal connection string. It is never stored in the repository.
- Migrations run automatically when the application starts.

## Verification
The deploy job retries a bounded number of times and fails unless both return 200.
To check by hand after a run:

    curl -s -o /dev/null -w '%{http_code}\n' "$STAGING_URL/health"
    curl -s -o /dev/null -w '%{http_code}\n' "$STAGING_URL/items"

- `/health` 200: the process is alive.
- `/items` 200: the API can query PostgreSQL.

## Rollback
Every release is also tagged with its commit SHA, which never moves.
1. Pick the last good commit SHA from the Actions history.
2. In Render, Settings > Image URL, set
   `ghcr.io/sagalou/holbertonschool-hbtn-devops-pipeline-lab:<sha>` and save.
3. Trigger a manual deploy and run both verification requests.
4. Fix forward on `main`, then point the Image URL back to `:latest`.

## Cleanup
When the lab is over:
1. Delete the Render web service and the Render PostgreSQL database.
2. Revoke the Render API key in Render account settings.
3. Delete `RENDER_API_KEY`, `RENDER_SERVICE_ID` and `STAGING_URL` from the GitHub repository.
4. Set the GHCR package back to private, or delete it.
