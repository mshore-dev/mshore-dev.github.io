---
title: "Backups, eh?"
date: 2023-03-02
tags: ["self-hosting", "backups"]
---
## The importance of backups
Backups are important. No matter what kind of data you are storing, chances are you will want to have it backed up somewhere. Some data is inconvenient to lose, and some data is devastating to lose. If you have data of any real importance, it should be properly backed up. Hardware can last a long time, but one failure can leave you in quite a predicament.

## Backing up my important data
For the past couple of years, I have run a small NextCloud instance for me and a few of my friends. I cannot speak to my friends' use-case, but I keep relatively important data on my NextCloud instance. The data is not irreplaceable, the complete backup of my camera roll lives on my phone, and the previous semester's schoolwork is not of super high importance. However, losing that data would be a pain.

Although unlikely, the day my phone ceases to function could also be the day my hosting provider experiences a data-center fire. In this scenario, my data is *gone*. Even though it is highly unlikely this scenario will happen, it is not impossible. Therefore, it is important to take measures to ensure my data, and the data of my friends, is safely backed up.

## Actually backing up data
A quick disclaimer: I am not an expert on backup solutions. This is something I put together to fit my simple needs.

The first step was finding a tool to facilitate backups in such a way that I could search through the data by date. I specifically wanted to be able to pull files from a certain day's backup. Since a majority of my data on NextCloud was word documents, the ability to pull a specific version of the file was key. NextCloud does a pretty good job of keeping file versions, but this is an extra layer of security atop that.

For this, I landed on [restic](https://restic.net/). Restic is available across a wide variety of operating systems, and is natively compatible with several storage providers. It is also free and open-source software, which is always a plus. Restic handles backups by taking snapshots of the filesystem, de-duplicating, and encrypting the data. The backed-up data is then stored in a local "repository" that can be read from or added to as needed.

To facilitate creating backups in my Docker Compose environment, I created [a custom Docker image](https://github.com/mshore-dev/alpine-cron-backup) based on Alpine Linux, including Restic, RClone, and `crond`. By leveraging both Restic and RClone, I am able to have the backup exist both on the server, and on a remote object storage host. The logic of the backup is simple, with a few variations depending on what exactly is being backed up.

### NextCloud Backup

The NextCloud backup has the most steps, since it has a "maintenance mode" that can be enabled to ensure data consistency during the backup. The backup script first enables NextCloud's maintenance mode, then creates a dump of the PostgreSQL database. Afterwards, Restic is used to back up the files from NextCloud and the previously dumped database. Finally, maintenance mode is disabled and the backup repository is synced to Backblaze B2.

### Mastodon Backup

Mastodon, unlike NextCloud, does not have a maintenance mode. Otherwise, the logic follows the same flow, except with an extra database: Redis. Redis automatically syncs data to a `dump.rdb` on disk. That file is simply included in the folder to be backed up.

### lldap Backup

Last, but certainly not least, is [lldap](https://github.com/nitnelave/lldap). lldap is a light LDAP implementation I use to have my self-hosted services operate under a single set of credentials. lldap stores its data in an SQLite3 database, so backing it up is a breeze.

## Now you've got a backup!

Hurray! Now I have a functioning backup system. It has been running for the past month without issue. However, there is one final issue: restoring from the backup. Thankfully, I have not yet had an incident requiring this. However, a backup does no good if you cannot restore from it. Preliminary testing has proven data can be retrieved from the backup, but I have not yet tried it in a "real-world scenario". Over the next few weeks, I intend to investigate and come up with a procedure for restoring data, in the event it is required.
