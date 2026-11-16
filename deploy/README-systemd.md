# VPS deployment without Docker (systemd)

Prefer this over `deploy/README.md` (Docker) if you'd rather manage the
processes directly with systemd — same 24/7 reliability, one less layer.

## 1. Create a dedicated user and directory

```bash
sudo useradd --system --create-home --shell /bin/bash inboxops
sudo mkdir -p /opt/inboxops
sudo chown inboxops:inboxops /opt/inboxops
```

## 2. Clone and set up as that user

```bash
sudo -u inboxops -i
cd /opt/inboxops
git clone https://github.com/Supriya5784/indexops.git .

python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt

cp .env.example .env
nano .env   # set ENVIRONMENT=production, API_KEY, POSTGRES_PASSWORD/DATABASE_URL, Telegram, etc.

# Postgres (if not already installed):
sudo apt install postgresql
sudo -u postgres createuser inboxops
sudo -u postgres createdb -O inboxops inboxops
sudo -u postgres psql -c "ALTER USER inboxops PASSWORD 'yourpassword';"
# then set DATABASE_URL=postgresql://inboxops:yourpassword@localhost/inboxops in .env

alembic upgrade head

python -m app.gmail.auth   # one-time OAuth2 flow, needs credentials.json present
deactivate
exit   # back to your normal user
```

## 3. Dashboard setup

```bash
sudo -u inboxops -i
cd /opt/inboxops/dashboard
npm install
npm run build
cp .env.local.example .env.local
nano .env.local   # BACKEND_API_URL=http://localhost:8000, BACKEND_API_KEY=<same as API_KEY above>
exit
```

## 4. Install the systemd units

```bash
sudo cp deploy/inboxops-backend.service /etc/systemd/system/
sudo cp deploy/inboxops-dashboard.service /etc/systemd/system/
sudo cp deploy/inboxops-prune-logs.service /etc/systemd/system/
sudo cp deploy/inboxops-prune-logs.timer /etc/systemd/system/

sudo systemctl daemon-reload
sudo systemctl enable --now inboxops-backend
sudo systemctl enable --now inboxops-dashboard
sudo systemctl enable --now inboxops-prune-logs.timer
```

## 5. Check status / logs

```bash
sudo systemctl status inboxops-backend
sudo journalctl -u inboxops-backend -f
sudo journalctl -u inboxops-dashboard -f
```

## 6. Nginx + HTTPS

Same nginx config and certbot steps as `deploy/README.md` §4, except point
`proxy_pass` at `http://127.0.0.1:3000` (dashboard) and
`http://127.0.0.1:8000` (backend) instead of Docker service names.

## 7. Updating

```bash
sudo -u inboxops -i
cd /opt/inboxops
git pull
source .venv/bin/activate && pip install -r requirements.txt && alembic upgrade head && deactivate
cd dashboard && npm install && npm run build
exit

sudo systemctl restart inboxops-backend inboxops-dashboard
```
