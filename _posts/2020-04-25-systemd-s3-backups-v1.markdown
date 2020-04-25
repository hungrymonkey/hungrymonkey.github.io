---
layout: post
title: Systemd S3 Backups v1
date: 2020-4-25 00:00:00 +0300
description: You’ll find this post in your `_posts` directory. Go ahead and edit it and re-build the site to see your changes. # Add post description (optional)
img: 2020-04-25-systemd-dark-logo.svg # Add image post (optional)
tags: [s3, aws, backup, systemd, iam] # add tag
---

# Introduction

Welcome server admins!

My second post will discuss crafting backup jobs with systemd and aws s3 buckets.

## Dependencies
- awscli
- systemd
 
## Implementation

### Create the backup environment

1. Create the bucket
	- `aws s3 mb s3://<backup_bucket>`

2. Attach Required Access Control List

* For backups, awscli uses these permissions to upload objects. `s3:PutObject` allows IAM user or IAM role to write objects to the s3 bucket. Similar to unix write permission, subsequent calls to the same object will override it. `s3:AbortMultipartUpload` allows utilities to upload objects larger than 100 MB by permitting any s3 compatible tool to break upload into chunks. For directory copies, `aws sync` uses `s3:ListBucket` to scan and upload multiple files.

* If the source server is an EC2 instance, IAM admin can create an attachable role to allow credential-free s3 access. 

	- `aws iam create-role --role-name s3-<backup_bucket>-backup-ec2 --assume-role-policy-document file://iam-policy_backup-acl.json`

* If not, IAM admin can create an iam-user with these permissions below
	- `aws iam create-user --user-name <backup_bucket>-backup`
	- `aws iam create-policy --policy-name  iam-policy_backup-acl --policy-document=file://iam-policy_backup-acl.json`
	- `aws iam attach-user-policy --user-name  <backup_bucket>-backup  --policy-arn=arn:aws:iam::<aws_account_id>:user/<backup_bucket>-backup`

iam-policy_backup-acl.json

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor1",
            "Effect": "Allow",
            "Action": "s3:ListBucket",
            "Resource": "arn:aws:s3:::<backup_bucket>",
            "Condition": {
                "ForAnyValue:IpAddress": {
                    "aws:SourceIp": [
                        "<Restrict-IP>"
                    ]
                }
            }
        },
        {
            "Sid": "VisualEditor3",
            "Effect": "Allow",
            "Action": [
                "s3:PutObject",
                "s3:AbortMultipartUpload"
            ],
            "Resource": [
                "arn:aws:s3:::<backup_bucket>/prefix/*",
                "arn:aws:s3:::<backup_bucket>/prefix"
            ],
            "Condition": {
                "IpAddress": {
                    "aws:SourceIp": "<Restrict-IP>"
                }
            }
        }
    ]
}
```
Restore needs an additional `s3:GetObject` permission to access backups. Creation commands are similar to backup IAM policy or IAM user.

iam-policy_restore-acl.json

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor1",
            "Effect": "Allow",
            "Action": "s3:ListBucket",
            "Resource": "arn:aws:s3:::<backup_bucket>",
            "Condition": {
                "ForAnyValue:IpAddress": {
                    "aws:SourceIp": [
                        "<Restrict-IP>"
                    ]
                }
            }
        },
        {
            "Sid": "VisualEditor3",
            "Effect": "Allow",
            "Action": [
                "s3:PutObject",
                "s3:GetObject",
                "s3:AbortMultipartUpload"
            ],
            "Resource": [
                "arn:aws:s3:::<backup_bucket>/prefix/*",
                "arn:aws:s3:::<backup_bucket>/prefix"
            ],
            "Condition": {
                "IpAddress": {
                    "aws:SourceIp": "<Restrict-IP>"
                }
            }
        }
    ]
}
```

### Design a Systemd Backup Service

Systemd provides numerous facilities to decrease fragile boilerplate in comparison to classical sysv scripts. However, systemd places more restrictions and many services may needs extra options for backup.

http://0pointer.de/blog/projects/systemd-for-admins-3.html

1.  Shell command lines are not supported
This syntax is inspired by shell syntax, but only the meta-characters and expansions described in the following paragraphs are understood, and the expansion of variables is different. Specifically, redirection using `<`, `<<`, `>`, and `>>`, pipes using `|`, running programs in the background using `&`, and other elements of shell syntax are not supported.

Solution: Run the command in a subshell such that /bin/sh -c ‘command | command`. Systemd requires all commands to be call by the absolute path.
```
ExecStart=/bin/sh -c 'dmesg | tac'
```

2. Remote transfer requires login credentials.

	s3-backup.service
	
	```
	[Unit]
	Description=AWS IAM user Backup
	
	[Service]
	Environment=AWS_ACCESS_KEY_ID=<iam-user-access-id>
	Environment=AWS_SECRET_ACCESS_KEY=<iam-user-secret-key>
	Type=oneshot
	ExecStart=/bin/sh -c 'backup create && aws s3 sync /home/user-data/backup/ s3://<backup_bucket>/prefix/$$(date +%%m-%%d-%%Y) --exclude "cache/*"'
	User=root
	Group=systemd-journal
	```

3. Daemon needs to be stopped before backup.


	Some services, such as rocketchat, require the server to be down before the backup routine can run. With `ExecStartPre`, systemd.service run a command before `ExecStart`. `oneshot` allows multiple `ExecStartPre` in order.
	
	However, `ExecStartPre` may fail and cause systemd to ignore `ExecStart` as describe in man page,  “ExecStart= commands are only run after all ExecStartPre= commands that were not prefixed with a `-` exit successfully”. To start the server regardless, all backup commands will be prefixed with `-` to ignore the result, so `ExecStart` can start the server.
	
	As a downside to extending `ExecStartPre`, many backup commands will not be passed to `journalctl` when the service succeeds. It might be preferable to create a subshell in `ExecStart` instead.


	s3-backup.service
	
	```
	[Unit]
	Description=Backup service for Rocketchat
	
	[Service]
	Environment=ROCKETCHAT_BACKUP_DIR=/var/snap/rocketchat-server/common/backup
	Type=oneshot
	ExecStartPre=/usr/bin/sudo /usr/sbin/service snap.rocketchat-server.rocketchat-server stop
	ExecStartPre=-/usr/bin/sudo /usr/bin/snap run rocketchat-server.backupdb
	ExecStartPre=-/usr/bin/sudo /bin/sh -c 'aws s3 sync ${ROCKETCHAT_BACKUP_DIR} s3://backup_bucket/rocketchat/$$(date +%%m-%%d-%%Y)/'
	ExecStart=/usr/sbin/service snap.rocketchat-server.rocketchat-server start
	User=ubuntu
	Group=systemd-journal
	```


4. Avoid Writing to the File System

	Whenever anyone wants to avoid the filesystem altogether, pipe the output to STDIN and upload it to a s3 bucket.
	
	s3-backup.service
	
	```
	[Unit]
	Description=Backup service for Matrix Synapse
	
	[Service]
	Environment=AWS_BUCKET=s3://<backup_bucket/prefix
	Type=oneshot
	ExecStartPre= > /postgres.sql.gz'
	ExecStart=/bin/sh -c 'docker run --rm --network=matrix \
	                --env-file=/matrix/postgres/env-postgres-psql \
	                postgres:12.1-alpine \
	pg_dumpall -h matrix-postgres | gzip -c  | \
	aws s3 - /postgres.sql.gz ${AWS_BUCKET}/$$(date +%%m-%%d-%%Y)/postgres.sql.gz'
	User=root
	Group=systemd-journal
	```
5. Remove Generated Backup Files

	To preserve space, systemd can remove generated files. However, `ExecStartPre` resolves globs on startup such that `*` will not see files generated by another `ExecStartPre`. In order to resolve this issue, we move the `rm` into `ExecStart` and use `;` to ensure rm run regardless of upload status.
	
	Systemd verify will complain about removing files on the disk

	```
	# systemd-analyze verify s3-backup.service
	Attempted to remove disk file system, and we can't allow that.
	```

	s3-backup.service
	
	```
	[Unit]
	Description=Mailinabox backup service
	
	[Service]
	Environment=AWS_BUCKET=s3://<backup_bucket/prefix
	Type=oneshot
	ExecStartPre=/bin/sh -c '/home/ubuntu/mailinabox/management/backup.py backup create’
	ExecStart=/bin/sh -c ‘aws s3 sync /home/user-data/backup/ ${AWS_BUCKET}/$$(date +%%m-%%d-%%Y); rm /home/user-data/backup/encrypted/*’
	User=root
	Group=systemd-journal%
	```

### Decide when to schedule systemd timers. 

Systemd imposes little to no restrictions on scheduling tasks for backups. Please refer to the `man systemd.time`. As long as the timer and service have the same name, `systemctl enable s3-backup.timer` will find and link the correct service.
	
	
s3-backup.timer
	
```
[Unit]
Description=Backup timer for any service named s3-backup.service
	
[Timer]
OnCalendar=Sun,Tue,Thu,Sat 02:00
Persistent=true
	
[Install]
WantedBy=timers.target
```

## Helpful Commands
1. Reload the daemon after any modification
	- `# systemctl reload-daemon`
	
2. Verify unit file grammar
	- `# systemd-analyze verify s3-backup.timer`

3. View Backup status

   a. View systemctl status

	```
	$ systemctl status s3-backup.service
	● s3-backup.service - Backup service for Rocketchat
	   Loaded: loaded (/etc/systemd/system/s3-backup.service; static; vendor preset: enabled)
	   Active: inactive (dead) since Sat 2020-04-25 14:29:03 UTC; 50min ago
	  Process: 1525 ExecStart=/usr/bin/sudo /bin/sh -c /bin/rm ${ROCKETCHAT_BACKUP_DIR)/*; /usr/sbin/service snap.rocketchat-server.rocketchat-server start (code=exited, status=0/SUCCESS)
	  Process: 1504 ExecStartPre=/usr/bin/sudo /bin/sh -c aws s3 sync ${ROCKETCHAT_BACKUP_DIR} s3://backup_bucket/rocketchat/$$(date +%m-%d-%Y)/ (code=exited, status=0/SUCCESS)
	  Process: 1449 ExecStartPre=/usr/bin/sudo /usr/bin/snap run rocketchat-server.backupdb (code=exited, status=0/SUCCESS)
	  Process: 1404 ExecStartPre=/usr/bin/sudo /usr/sbin/service snap.rocketchat-server.rocketchat-server stop (code=exited, status=0/SUCCESS)
	 Main PID: 1525 (code=exited, status=0/SUCCESS)
	
	Apr 25 14:28:58 ip-172-31-16-131 sudo[1504]:   ubuntu : TTY=unknown ; PWD=/ ; USER=root ; COMMAND=/bin/sh -c aws s3 sync /var/snap/rocketchat-server/common/backup/ s3://backup_bucket/rocketchat/$(da
	Apr 25 14:28:58 ip-172-31-16-131 sudo[1504]: pam_unix(sudo:session): session opened for user root by (uid=0)
	Apr 25 14:28:59 ip-172-31-16-131 sudo[1504]: [214B blob data]
	Apr 25 14:29:01 ip-172-31-16-131 sudo[1504]: [48.0K blob data]
	Apr 25 14:29:02 ip-172-31-16-131 sudo[1504]: [21.5K blob data]
	Apr 25 14:29:02 ip-172-31-16-131 sudo[1504]: pam_unix(sudo:session): session closed for user root
	Apr 25 14:29:02 ip-172-31-16-131 sudo[1525]:   ubuntu : TTY=unknown ; PWD=/ ; USER=root ; COMMAND=/bin/sh -c /bin/rm /var/snap/rocketchat-server/common/backup/*; /usr/sbin/service snap.rocketchat-server
	Apr 25 14:29:02 ip-172-31-16-131 sudo[1525]: pam_unix(sudo:session): session opened for user root by (uid=0)
	Apr 25 14:29:03 ip-172-31-16-131 sudo[1525]: pam_unix(sudo:session): session closed for user root
	Apr 25 14:29:03 ip-172-31-16-131 systemd[1]: Started Backup service for Rocketchat.
	```

	b. View Backup Logs

	```
	$ journalctl -u s3-backup.service
	Apr 25 14:28:40 ip-172-31-16-131 systemd[1]: Starting Backup service for Rocketchat...
	Apr 25 14:28:40 ip-172-31-16-131 sudo[1404]:   ubuntu : TTY=unknown ; PWD=/ ; USER=root ; COMMAND=/usr/sbin/service snap.rocketchat-server.rocketchat-server stop
	Apr 25 14:28:40 ip-172-31-16-131 sudo[1404]: pam_unix(sudo:session): session opened for user root by (uid=0)
	Apr 25 14:28:41 ip-172-31-16-131 sudo[1404]: pam_unix(sudo:session): session closed for user root
	Apr 25 14:28:41 ip-172-31-16-131 sudo[1449]:   ubuntu : TTY=unknown ; PWD=/ ; USER=root ; COMMAND=/usr/bin/snap run rocketchat-server.backupdb
	Apr 25 14:28:41 ip-172-31-16-131 sudo[1449]: pam_unix(sudo:session): session opened for user root by (uid=0)
	Apr 25 14:28:42 ip-172-31-16-131 sudo[1449]: [*] Creating backup file...
	Apr 25 14:28:58 ip-172-31-16-131 sudo[1449]: [+] A backup of your data can be found at /var/snap/rocketchat-server/common/backup/rocketchat_backup_20200425.1428.tar.gz
	Apr 25 14:28:58 ip-172-31-16-131 sudo[1449]: pam_unix(sudo:session): session closed for user root
	Apr 25 14:28:58 ip-172-31-16-131 sudo[1504]:   ubuntu : TTY=unknown ; PWD=/ ; USER=root ; COMMAND=/bin/sh -c aws s3 sync /var/snap/rocketchat-server/common/backup/ s3://mederrata-backups/rocketchat/$(da
	Apr 25 14:28:58 ip-172-31-16-131 sudo[1504]: pam_unix(sudo:session): session opened for user root by (uid=0)
	Apr 25 14:28:59 ip-172-31-16-131 sudo[1504]: [214B blob data]
	Apr 25 14:29:01 ip-172-31-16-131 sudo[1504]: [48.0K blob data]
	Apr 25 14:29:02 ip-172-31-16-131 sudo[1504]: [21.5K blob data]
	Apr 25 14:29:02 ip-172-31-16-131 sudo[1504]: pam_unix(sudo:session): session closed for user root
	Apr 25 14:29:02 ip-172-31-16-131 sudo[1525]:   ubuntu : TTY=unknown ; PWD=/ ; USER=root ; COMMAND=/bin/sh -c /bin/rm /var/snap/rocketchat-server/common/backup/*; /usr/sbin/service snap.rocketchat-server
	Apr 25 14:29:02 ip-172-31-16-131 sudo[1525]: pam_unix(sudo:session): session opened for user root by (uid=0)
	Apr 25 14:29:03 ip-172-31-16-131 sudo[1525]: pam_unix(sudo:session): session closed for user root
	Apr 25 14:29:03 ip-172-31-16-131 systemd[1]: Started Backup service for Rocketchat.
	```
4. Start the timer
	- `#  systemctl enable s3-backup.timer`
	- `# systemctl start s3-backup.timer`
5. Run Backup Routine right now
	- `# systemctl start s3-backup.service` 

## TODO
- Email failures
- Allow backup commands appear in journalctl logs
- Forward exit status

## Links
- https://www.freedesktop.org/software/systemd/man/systemd.time.html
- http://0pointer.de/public/systemd-man/systemd.service.html
- https://www.freedesktop.org/software/systemd/man/systemd.service.html
- https://www.freedesktop.org/software/systemd/man/systemd.timer.html
- https://wiki.archlinux.org/index.php/Systemd/Timers

