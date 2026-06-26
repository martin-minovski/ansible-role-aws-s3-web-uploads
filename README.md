An Ansible role that mirrors IDE uploads from S3 into a host web directory. Developers push files with `aws s3 cp` or `aws s3 sync` under a per-host prefix; S3 event notifications enqueue work on SQS; an on-host daemon applies incremental changes and a companion inotify watcher keeps ownership and permissions compatible with nginx/php-fpm.

Upload scope is enforced in IAM: each developer policy should only allow `s3://<bucket>/<fqdn>/...`.

## Requirements

- Ansible 11 or 12
- `ansible.posix` collection (`ansible-galaxy collection install -r requirements.yml`)
- `awscli`, `jq`, and `inotify-tools` (installed by the role)
- EC2 instance profile (or equivalent) with S3 read and SQS consume/delete on the uploads bucket and queue

## Role variables

### Required (set in your playbook or inventory)

| Variable | Description |
|----------|-------------|
| `fqdn` | Host identifier; used for the S3 prefix and systemd instance name |
| `uploads_bucket` | S3 bucket receiving IDE uploads |
| `uploads_queue_url` | SQS queue fed by S3 object notifications |

### Optional

| Variable | Default | Description |
|----------|---------|-------------|
| `web_dir` | `/var/www/{{ fqdn }}` | Local directory nginx serves |
| `web_group` | `www-data` | Unix group for synced files (nginx/php-fpm) |
| `uploads_prefix` | `{{ fqdn }}/` | S3 key prefix for this host |
| `aws_region` | `eu-west-1` | AWS region for CLI calls |
| `sync_exclude_paths` | `vendor/*`, `node_modules/*` | Paths skipped by sync and SQS-driven deletes |
| `sqs_batch_size` | `10` | Max SQS messages per receive (1–10) |
| `sqs_wait_seconds` | `20` | SQS long-poll wait (0–20) |
| `perm_watcher_excludes` | `node_modules`, `vendor` | Dirs skipped by the inotify perm-watcher |
| `perm_watcher_enable_logging` | `false` | Verbose perm-watcher logging |
| `bin_dir` | `/usr/local/sbin` | Install path for helper scripts |
| `systemd_unit_dir` | `/etc/systemd/system` | Install path for unit templates |
| `uploads_run_baseline_sync` | `true` | Run `aws s3 sync` once at converge time |
| `uploads_manage_services` | `true` | Enable/start systemd units |

## Systemd units

The role installs templated units and enables instances named after `fqdn`:

- `s3-uploads-sync@{{ fqdn }}.service` — long-polls SQS, runs targeted `aws s3 cp` / `rm`, baseline sync at startup
- `web-perm-watcher@{{ fqdn }}.service` — inotify-based ownership/mode enforcement under `web_dir`

## Example

```yaml
- hosts: webservers
  roles:
    - role: aws_s3_web_uploads
      vars:
        fqdn: "{{ inventory_hostname }}"
        uploads_bucket: "{{ uploads_bucket }}"
        uploads_queue_url: "{{ uploads_queue_url }}"
        aws_region: eu-west-1
```

## License

Apache 2.0
