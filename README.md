# Tic Tac Toe CI/CD practice app

This repository contains a Node.js Tic Tac Toe app and the GitHub Actions
pipeline that tests and deploys it. The app is deployed to the existing EC2
location `/home/ubuntu/tech611-ttt-cicd` and is managed by PM2 as `ttt-app`.

## Run locally

Node.js 20 is the supported runtime.

```bash
cd app
npm ci --ignore-scripts
npm test
npm start
```

Open <http://localhost:3000>. `npm ci --ignore-scripts` is recommended for a
normal install because the package's `postinstall` script is a seeding utility,
not a build step. Run `npm run postinstall` explicitly only when you intend to
seed a running app or MongoDB instance.

The default test command runs the self-contained test suite and skips MongoDB
integration cases. To run everything locally, start MongoDB and use:

```bash
cd app
MONGODB_INTEGRATION_URI=mongodb://127.0.0.1:27017/tictactoe_local npm run test:all
```

The test helper automatically changes the database name to end in `_test`.
Never point `MONGODB_INTEGRATION_URI` at a database containing data you need.

## Pipeline

The workflow is [`.github/workflows/cicd.yml`](.github/workflows/cicd.yml).

- A pull request targeting `dev` runs unit and MongoDB integration tests. It
  does not merge or deploy.
- A push to `dev` runs the same tests using Node.js 20 and MongoDB 7.
- After a successful `dev` run, the exact tested commit is fast-forwarded to
  `main`. Promotion fails safely if `main` has diverged.
- Only a successfully promoted commit is copied to the existing EC2 directory.
  Production dependencies are installed, the existing `ttt-app` PM2 process is
  restarted (or created), and the runner checks the app on port 3000.
- Concurrency prevents overlapping runs for `dev`, which protects the shared
  `main` branch and EC2 deployment.

Configure branch protection so pull requests into `dev` require the
`Test (Node.js 20)` check. Changes pushed directly to `main` are intentionally
not deployed; `dev` is the release branch.

## GitHub Actions secrets

The deploy job requires these repository or organization secrets:

| Secret | Purpose |
| --- | --- |
| `EC2_HOST` | EC2 hostname or IP address |
| `EC2_USERNAME` | SSH user for the existing host |
| `EC2_SSH_KEY` | Private SSH key accepted by the host |
| `EC2_SSH_KNOWN_HOSTS` | Trusted SSH host-key entry for the EC2 host |

Create `EC2_SSH_KNOWN_HOSTS` from a host key obtained through a trusted channel
(for example, from the server administrator). Do not discover and trust the key
inside the workflow: that would not protect the deployment from a
man-in-the-middle attack.

The existing EC2 host must have Node.js 20, npm, PM2, and `curl` available to
the SSH user. The GitHub-hosted runner supplies `rsync`; the EC2 SSH user must
be able to write to `/home/ubuntu/tech611-ttt-cicd` and manage the `ttt-app`
PM2 process.

The GitHub Actions token needs read access for tests and write access to
repository contents for the `promote` job. The workflow grants write access
only to that job. If organization policy disables workflow write tokens, allow
GitHub Actions to write repository contents or replace automatic promotion with
the organization's approved mechanism.

## Application configuration

All settings are optional unless your deployment depends on the corresponding
feature.

| Variable | Default | Meaning |
| --- | --- | --- |
| `PORT` | `3000` | HTTP listen port; the deployment smoke test expects 3000 |
| `STATEFUL_MODE` | client-local | Set to `server` or `true` for server-memory fallback state |
| `MONGODB_URI` | unset | Enables persistent MongoDB-backed state |
| `USE_MARKER_IMAGES` | `false` | Set to `true` to render image markers |
| `MARKER_IMAGES_PATH` | `/images` | HTTP(S) base URL for marker images |
| `APP_FOOTER_TIMESTAMP` | unset | Text appended to the footer version |
| `LOG_LEVEL` | `info` | Pino log level |
| `LOG_DESTINATION` | `stdout` | Set to `file` to write logs to disk |
| `LOG_FILE_PATH` | `./logs/app.log` | Log path when file logging is enabled |

Runtime variables are managed on the EC2 host/within the existing PM2 process;
the workflow does not replace them. `pm2 restart --update-env` preserves the
current deployment approach while applying the host's configured environment.
