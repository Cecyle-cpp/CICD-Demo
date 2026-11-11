# CI/CD Pipeline: GitHub → ngrok → Jenkins (Docker) → Webex

Commit → GitHub webhook → ngrok → Jenkins build + pytest → Webex bot message.

## Repo layout

```
calculator.py          # demo Python code
test_calculator.py     # pytest unit tests
requirements.txt
Jenkinsfile            # declarative pipeline (checkout, venv, pytest, Webex notify)
jenkins/Dockerfile     # Jenkins LTS + python3/venv + plugins
jenkins/docker-compose.yml
```

## 1. GitHub repo

```bash
cd cicd-demo
git init -b main
git add .
git commit -m "Initial commit"
# create an empty repo on github.com first, then:
git remote add origin https://github.com/<you>/<repo>.git
git push -u origin main
```

## 2. Run Jenkins in Docker

```bash
cd jenkins
docker compose up -d --build
docker logs jenkins          # copy the initial admin password
```

Open http://localhost:8080, paste the password, choose **Select plugins to install → None** (the Dockerfile already installed what's needed) or Install suggested plugins, then create your admin user.

Confirm under **Manage Jenkins → Plugins → Installed** that these are present: Git, GitHub, Pipeline, JUnit, Credentials Binding.

## 3. Expose Jenkins with ngrok

```bash
ngrok config add-authtoken <your-token>   # one time
ngrok http 8080
```

Copy the `https://xxxx.ngrok-free.app` forwarding URL. Keep this terminal visible: the webhook shows up here as `POST /github-webhook/ 200` (20 pts in the recording).

Optional: set **Manage Jenkins → System → Jenkins URL** to the ngrok URL. Note the free ngrok URL changes each time you restart, so update the GitHub webhook if it does.

## 4. GitHub webhook

Repo → **Settings → Webhooks → Add webhook**

- Payload URL: `https://xxxx.ngrok-free.app/github-webhook/` (trailing slash required)
- Content type: `application/json`
- Events: **Just the push event**
- Active: checked

After saving, the "Recent Deliveries" tab should show a green check (the first ping).

## 5. Webex bot

1. Go to https://developer.webex.com/my-apps → **Create a New App → Create a Bot**. Name it, pick an icon, and copy the **Bot access token** (shown once).
2. In the Webex app, create a space and **add the bot** (search by its bot username).
3. Get the space's room ID. Easiest way:

   ```bash
   curl -s -H "Authorization: Bearer <BOT_TOKEN>" https://webexapis.com/v1/rooms | python3 -m json.tool
   ```

   Copy the `id` of your space. (The bot only lists spaces it belongs to.)
4. Quick test:

   ```bash
   curl -s -X POST https://webexapis.com/v1/messages \
     -H "Authorization: Bearer <BOT_TOKEN>" -H "Content-Type: application/json" \
     -d '{"roomId":"<ROOM_ID>","markdown":"Hello from my bot"}'
   ```

## 6. Jenkins credentials

**Manage Jenkins → Credentials → System → Global → Add Credentials**

| Kind        | ID                | Secret                |
|-------------|-------------------|-----------------------|
| Secret text | `webex-bot-token` | the bot access token  |
| Secret text | `webex-room-id`   | the space's room ID   |

The IDs must match the `Jenkinsfile` exactly.

## 7. Jenkins job

**New Item → Pipeline** (name e.g. `cicd-demo`)

- **Build Triggers**: check **GitHub hook trigger for GITScm polling**
- **Pipeline**: Definition = *Pipeline script from SCM*, SCM = Git, Repository URL = your GitHub repo URL, Branch = `*/main`, Script Path = `Jenkinsfile`
  - If the repo is private, add a GitHub username + personal access token credential.
- Save, then click **Build Now** once. This registers the trigger and confirms Python, pytest and Webex all work before you record.

## 8. Test end to end / recording checklist

Arrange windows first: terminal with ngrok, browser with GitHub repo, browser with Jenkins job, Webex space.

1. Edit code (e.g. add a `power(a, b)` function plus a test, or change a test value), then:
   ```bash
   git add . && git commit -m "Add power function" && git push
   ```
2. **GitHub**: refresh the repo page and show the commit with "now" as the timestamp. (10 pts)
3. **ngrok terminal**: show the incoming `POST /github-webhook/` line with a 200 response. (20 pts)
4. **Jenkins**: show the build starting on its own, with "Started by GitHub push". (20 pts)
5. **Console Output**: open it and scroll to show all pytest tests `PASSED` and `Finished: SUCCESS`. (20 pts)
6. **Webex**: show the bot's build status message arriving in the space. (10 pts)

Bonus idea: push a deliberately failing test afterward to show the ❌ FAILURE notification.

## Troubleshooting

- **Webhook shows 403 or no build**: check the URL ends with `/github-webhook/`, and that Jenkins' security settings don't block anonymous webhook calls (CSRF protection is fine; the GitHub plugin handles it).
- **`python3: not found` / `venv` error in the build**: Jenkins was started from the stock image; rebuild with `docker compose up -d --build`.
- **Webex message not sent**: check the credential IDs, that the bot is in the space, and the curl output in the console log (a 404 usually means a wrong room ID).
- **ngrok URL changed**: update the webhook Payload URL in GitHub.
