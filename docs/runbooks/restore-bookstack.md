Runbook: restore BookStack from backup

Use this when: BookStack's database is lost or damaged (the wiki shows errors, pages are missing, or the stack_bookstack_db volume was deleted). Impact while running: the wiki is down. Expect about 10 minutes. Data loss: everything written since the last backup (nightly at about 03:30). Check the snapshot time in step 2.

Last drill: 10/30/2026, took 8 minutes, by Soviet. Update this line after every test.

What you need before you start
SSH access to apps, and the ability to become deploy (sudo -iu deploy).
Garage must be reachable: ops is up and s3 answers from apps.

How it works

The nightly backup contains bookstack.sql: a text file of SQL commands that rebuilds every table and row. Restoring means: start an empty database, extract that file from the encrypted backup, and replay it into the database.

BookStack's uploaded images and settings live in a separate volume (stack_bookstack_config). This runbook restores the database only

Steps

All commands run on apps as deploy, inside /opt/selfhosted-stack, unless stated otherwise.

1. Become deploy and go to the stack folder
bash
# Run on: apps
sudo -iu deploy
# Run on: apps (as deploy)
cd /opt/selfhosted-stack

2. Check which backups exist
bash
docker compose --profile backup run --rm --entrypoint restic stack-backup \
  -o s3.bucket-lookup=path snapshots

The list shows each snapshot's ID, time and tags. The newest one is latest. If you need an older one (for example the newest is damaged), note its ID and use it instead of latest in step 5.

3. Stop BookStack and its database
bash
docker compose stop bookstack bookstack-db

Only if the database volume is damaged or gone (skip this if you are just rolling back data), delete it so a clean one can be created. This destroys the current database for good:

bash
docker compose rm -f bookstack-db
docker volume rm stack_bookstack_db

4. Recreate an empty database
bash
docker compose up -d bookstack-db
docker compose ps bookstack-db

Wait until the status says healthy. MariaDB creates the bookstack database and user from the settings in compose.yaml on its first start.

5. Extract the SQL file from the backup
bash
mkdir -p ~/restore
docker compose --profile backup run --rm -v "$HOME/restore:/restore" \
  --entrypoint restic stack-backup -o s3.bucket-lookup=path \
  restore latest --target /restore --include /tmp/backup/bookstack.sql
ls -l ~/restore/tmp/backup/bookstack.sql

Replace latest with a snapshot ID to use an older backup. The file must exist and not be empty. Note the option is spelled s3.bucket-lookup with a dot.

6. Load it into the database
bash
docker compose exec -T bookstack-db sh -c 'mariadb -uroot -p"$MARIADB_ROOT_PASSWORD"' \
  < ~/restore/tmp/backup/bookstack.sql

No output means success. Keep the single quotes: the password variable must be read inside the container.

7. Start BookStack
bash
docker compose up -d bookstack
docker compose logs --tail 20 bookstack
Check that it worked

Open https://wiki.ca and log in.
Your pages are there
Uptime Kuma shows wiki. green again.

Clean up

The restored file is an unencrypted copy of the whole wiki. Delete it (root owns it):

bash
# Run on: apps (as deploy)
exit
# Run on: apps
sudo rm -rf /home/deploy/restore

If something fails
Symptom	Cause	Fix
wrong password or no key found	RESTIC_PASSWORD in .env is not the one the repository was created with	Use the original password from your password manager. A repository cannot be opened with another one
Restic cannot connect	ops or Garage is down, or the s3. allowlist blocks apps	On ops: docker compose ps. From apps: curl -sS -i https://s3.domain.ca should return a 403 with an XML body
mariadb: Access denied in step 6	The database was not recreated empty	Redo step 3 (delete the volume) and step 4
The file in step 5 is missing	Wrong snapshot, or --include path not in it	Run restic ls SNAPSHOT_ID through the same command to see the paths
BookStack shows an error page after restore	Wrong BOOKSTACK_DB_PASSWORD in .env	docker compose logs bookstack; fix .env, then docker compose up -d bookstack
