# hello-world-ci-cd

Minimal "Hello World" web page, containerized, auto-deployed to a free-tier
AWS EC2 instance by a locally-hosted Jenkins whenever `main` on GitHub changes.

```
push to GitHub main --> local Jenkins (polls repo) --> SSH to EC2 --> docker compose up --build
```

## What's here

- [`index.html`](index.html) / [`Dockerfile`](Dockerfile) / [`docker-compose.yml`](docker-compose.yml) — the app itself: an nginx container serving one static page.
- [`Jenkinsfile`](Jenkinsfile) — the pipeline Jenkins runs on every change: checkout, then SSH into EC2 and rebuild/restart the container there.
- [`jenkins/docker-compose.yml`](jenkins/docker-compose.yml) — spins up Jenkins itself, locally, in Docker.

## 1. Create the free-tier EC2 instance (AWS Console)

You said AWS CLI isn't set up yet, so do this once in the console:

1. Sign in to the AWS Console → EC2 → **Launch instance**.
2. Name: `hello-world`.
3. AMI: **Amazon Linux 2023** (free tier eligible).
4. Instance type: **t2.micro** or **t3.micro** (both free-tier eligible — pick whichever the console marks "Free tier eligible").
5. Key pair: create a new one, e.g. `hello-world-key`, download the `.pem` file and keep it safe (never commit it — it's already in `.gitignore`).
6. Network settings → security group: allow
   - SSH (22) from **your IP only**
   - HTTP (80) from **Anywhere (0.0.0.0/0)**
7. Storage: leave the default 8 GB (within free tier).
8. Launch, then copy the instance's **public IPv4 address**.

## 2. Prep the EC2 box

SSH in and install Docker + git:

```bash
ssh -i hello-world-key.pem ec2-user@<EC2_PUBLIC_IP>

sudo dnf install -y docker git
sudo systemctl enable --now docker
sudo usermod -aG docker ec2-user
exit
# log back in so the docker group membership takes effect
ssh -i hello-world-key.pem ec2-user@<EC2_PUBLIC_IP>

git clone https://github.com/<your-gh-user>/hello-world-ci-cd.git
cd hello-world-ci-cd
docker compose up -d --build
```

Visit `http://<EC2_PUBLIC_IP>` — you should see "Hello World". Everything after
this point is Jenkins automating the `git pull && docker compose up -d --build`
you just did by hand.

## 3. Run Jenkins locally

```bash
cd jenkins
docker compose up -d
docker compose logs jenkins | grep -A2 "initialAdminPassword"
```

Open http://localhost:8080, paste the initial admin password, choose
**Install suggested plugins**, then also install the **SSH Agent** plugin
(Manage Jenkins → Plugins).

## 4. Add the EC2 SSH key as a Jenkins credential

Manage Jenkins → Credentials → System → Global credentials → **Add credentials**:

- Kind: **SSH Username with private key**
- ID: `ec2-ssh-key` (must match the `Jenkinsfile`)
- Username: `ec2-user`
- Private key: paste the contents of `hello-world-key.pem`

## 5. Point the Jenkinsfile at your EC2 box

Edit [`Jenkinsfile`](Jenkinsfile) and replace `YOUR_EC2_PUBLIC_IP` with the
instance's actual public IP (or, better, an Elastic IP so it doesn't change
on reboot), then commit and push.

## 6. Create the Jenkins pipeline job

New Item → **Pipeline** → name it `hello-world-ci-cd`:

- Pipeline → Definition: **Pipeline script from SCM**
- SCM: **Git**, repository URL: your GitHub repo
- Branch: `*/main`
- Script Path: `Jenkinsfile`
- Save, then **Build Now** once to confirm it works end to end.

The `Jenkinsfile` polls GitHub every 2 minutes (`pollSCM('H/2 * * * *')`) —
since this Jenkins runs on your machine with no public URL, a real GitHub
webhook can't reach it. If you later expose Jenkins publicly (e.g. via
`ngrok http 8080` or a reverse proxy), swap the `pollSCM` trigger for
`githubPush()` and add a webhook in the GitHub repo settings pointing at
`http://<public-url>/github-webhook/` for instant, push-triggered builds.

## 7. Try it end to end

Change the text in `index.html`, commit, push to `main`. Within ~2 minutes
Jenkins picks it up, SSHes into EC2, and rebuilds the container. Refresh
`http://<EC2_PUBLIC_IP>` to see the change.
