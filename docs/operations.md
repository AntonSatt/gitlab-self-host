# Operations runbook

These commands assume the repository is installed on the GitLab host and the
container is named `gitlab`.

## Routine checks

```bash
docker compose ps
docker exec gitlab gitlab-ctl status
docker exec gitlab cat /opt/gitlab/embedded/service/gitlab-rails/VERSION
docker exec gitlab gitlab-rake gitlab:check SANITIZE=true
curl -I https://git.example.com/users/sign_in
```

For the K3s route:

```bash
kubectl -n apps get endpointslice gitlab-docker
kubectl -n apps get service gitlab-docker
kubectl -n apps get ingress gitlab
```

## Backups

An application backup does not include `gitlab-secrets.json`, configuration,
TLS keys, SSH host keys, object storage, or other host files. Back up the
application and configuration separately, then copy both off the server.

```bash
docker exec gitlab gitlab-backup create
docker exec gitlab gitlab-ctl backup-etc

find data/backups -maxdepth 1 -type f -printf '%TY-%Tm-%Td %TH:%TM %s %f\n'
find config/config_backup -maxdepth 1 -type f -printf '%TY-%Tm-%Td %TH:%TM %s %f\n'
```

Verify that the new files are non-empty and calculate checksums before an
upgrade:

```bash
sha256sum data/backups/*_gitlab_backup.tar
sha256sum config/config_backup/gitlab_config_*.tar
```

Keep the configuration archive separate from the application backup because it
contains the keys that decrypt sensitive database values. A same-disk archive
is useful for rollback but is not a disaster-recovery backup.

## Upgrade procedure

Read the release notes for every version between the current and target release.
Use the latest patch of each required upgrade stop.

### 1. Run pre-upgrade checks

```bash
docker exec gitlab gitlab-rake gitlab:check SANITIZE=true
docker exec gitlab gitlab-rake gitlab:background_migrations:status
docker exec gitlab gitlab-rake gitlab:doctor:secrets
```

Every background migration must be `finished` or `finalized`. Resolve active,
paused, or failed migrations before continuing.

### 2. Back up

Create and verify both backups from the previous section. Also copy the current
Compose file:

```bash
cp docker-compose.yml docker-compose.pre-upgrade.yml
```

### 3. Pin and validate the target release

Change only `GITLAB_VERSION` in `.env`, then run:

```bash
docker compose config --quiet
docker compose pull gitlab
```

Pulling first avoids extending the downtime window.

### 4. Recreate GitLab

```bash
docker compose up -d gitlab
docker compose ps
docker compose logs -f gitlab
```

The container can report healthy before Puma has finished its first Rails boot.
Wait until the public sign-in page responds without 502.

### 5. Verify

```bash
docker exec gitlab cat /opt/gitlab/embedded/service/gitlab-rails/VERSION
docker exec gitlab gitlab-ctl status
docker exec gitlab gitlab-rake db:migrate:status | grep '^   down' || true
docker exec gitlab gitlab-rake gitlab:check SANITIZE=true
curl -I https://git.example.com/users/sign_in
git ls-remote https://git.example.com/group/public-project.git
```

Check background migrations again. New migrations can be queued by the update.
Let them finish before the next version change.

## Rollback

Do not reuse an application backup with a different GitLab version. Restore on
the exact CE or EE version that created the archive.

If the new container fails before database changes are applied, restore the old
image pin and recreate it:

```bash
cp docker-compose.pre-upgrade.yml docker-compose.yml
docker compose up -d gitlab
```

If database migrations ran, follow GitLab's Docker rollback and restore
documentation. Restore the application backup together with the matching
configuration and secrets archive.

- [Upgrade Docker instances](https://docs.gitlab.com/update/docker/)
- [Roll back a Docker instance](https://docs.gitlab.com/update/package/rollback/)
- [Restore GitLab](https://docs.gitlab.com/administration/backup_restore/restore_gitlab/)
