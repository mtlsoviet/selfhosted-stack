Runbook: restore Vaultwarden from backup

Use this when: the Vaultwarden vault is lost or damaged (the web vault shows errors, logins fail, or the stack_vaultwarden_data volume was deleted). Impact while running: passwords are unavailable. Expect about 10 minutes. Clients that cached their vault still work offline. Data loss: every change made since the last backup (nightly at about 03:30). Check the snapshot time in step 2.

Last drill: 30/10/2026, took 10 minutes, by Soviet. Update this line after every test.

What you need before you start
SSH access to apps, and the ability to become deploy (sudo -iu deploy).
The restic password and Garage key on apps.

Garage must be reachable: ops is up and s3.domain.ca answers from apps.
How it works

Vaultwarden stores everything in one volume:

What	Why it matters
db.sqlite3	The encrypted vault database
rsa_key.pem and other key files	Sign login sessions. Without them, clients must log in again and some features break
attachments/, icons/	File attachments and website icons

The nightly backup stores the files (/data/vaultwarden, excluding the live database) and a safe copy of the database (vaultwarden.sqlite3, made with SQLite's backup command). Restoring puts the files back, then puts the safe copy back under the name db.sqlite3. SQLite has no server, so unlike BookStack there is nothing to load into: it is a file copy.

Steps

All commands run on apps as deploy, inside /opt/selfhosted-stack, unless stated otherwise.

1. Become deploy and go to the stack folder
bash
Run on: apps
sudo -iu deploy
Run on: apps (as deploy)
cd /opt/selfhosted-stack

2. Check which backups exist
bash
docker compose --profile backup run --rm --entrypoint restic stack-backup \
  -o s3.bucket-lookup=path snapshots

Note the newest snapshot's time. If you need an older one, note its ID and replace latest with it in step 4.

3. Stop Vaultwarden
bash
docker compose stop vaultwarden

Never copy files into the volume while Vaultwarden runs: a database file copied under a running app can be corrupted.

Only if the volume is damaged or gone, delete it so a clean one is created (this destroys the current vault for good):

bash
docker compose rm -f vaultwarden
docker volume rm stack_vaultwarden_data

4. Restore the files and the database in one step
bash
docker compose --profile backup run --rm --entrypoint sh stack-backup -c '
set -eu
restic -o s3.bucket-lookup=path restore latest --target /restore \
  --include /data/vaultwarden --include /tmp/backup/vaultwarden.sqlite3
cp -a /restore/data/vaultwarden/. /data/vaultwarden/
cp /restore/tmp/backup/vaultwarden.sqlite3 /data/vaultwarden/db.sqlite3
sqlite3 /data/vaultwarden/db.sqlite3 "PRAGMA integrity_check;"
ls -la /data/vaultwarden
'

What it does, in order:

Decrypts the snapshot into a scratch folder inside the container.
Copies the app's files (rsa_key.pem, attachments, icons) into the real volume. -a keeps permissions and timestamps.
Copies the safe database copy into the volume as db.sqlite3.
Checks the database for damage. The output must be ok.
Lists the volume so you can see db.sqlite3 and rsa_key.pem are there.

Replace latest with a snapshot ID to use an older backup. Spell the option s3.bucket-lookup with a dot.

5. Start Vaultwarden
bash
docker compose up -d vaultwarden
docker compose logs --tail 20 vaultwarden

The log shows it starting and listening, with no errors.

Check that it worked
Open https://vault.domain.ca and log in with your account and master password.
Your items are there
Uptime Kuma shows vault. green again.
Open the Admin page (/admin) once to confirm your admin token still works.

Clean up

If you used a restore folder on the host, it holds unencrypted data. This runbook's step 4 keeps its scratch files inside the container, which Docker deletes. If you restored to a folder on apps by another method, delete it with sudo rm -rf as alex.

If something fails
Symptom	Cause	Fix
wrong password or no key found	RESTIC_PASSWORD in .env is not the one the repository was created with	Use the original password from your personal password manager
integrity_check prints anything but ok	That snapshot's database copy is damaged	Redo step 4 with an older snapshot ID from step 2
Login says the account does not exist	The restore did not land in the volume	Look at the ls -la output of step 4: db.sqlite3 must be there; rerun step 4
Logins work but attachments are missing	Only the database was restored	Rerun step 4 in full: it restores the files too
permission denied writing to /data/vaultwarden	The volume was mounted read-only in the backup service	In compose.yaml the vaultwarden_data line of stack-backup must have no :ro
Restic cannot connect	ops or Garage is down, or the s3. allowlist blocks apps	On ops: docker compose ps. From apps: curl -sS -i https://s3.domain.ca should return a 403 with an XML body
