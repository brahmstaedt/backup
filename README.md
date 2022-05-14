# Backup
My approach for backups from a Linux computer. For me to find this a quick reference, but also for everyone else who finds this useful. This is an opinionated approach and I do not guarantee that this will work for you.

## Strategy & Objectives
I like the 3-2-1 backup rule, which states that there there should always be 3 copies of your data, on two different devices with one copy being off-site. Of course, exceeding this approach will not hurt, but it also increases efforts and complexity.

In addition to that golden rule, here are eight more considerations:
1. Since all my primary instances of data are encrypted at rest, the backups on other devices should live up to that.
2. Backups should be taken constantly. Maybe not for every savepoint of a file but definitely more often than daily.
3. I want to be able to restore files immediately. Meaning: offline and without the need of physical access to another device.
4. Backups should run automatically in the background and should not require me to remember doing them manually every now and then.
5. On day X, I will not need a backup, I need a data restore. Only when backups are tested regularly, I can be sure to get that.
6. I prefer simple solutions, where a backup is also just a bunch of files. A proprietary blob that can only be interpreted by a specific backup tool version is a nightmare in the making.
7. I want to be able to store off-site copies on cheap public clouds without having to trust in their confidentiality, integrity, or availability.
8. The user account I work with on a daily basis shall be able to add data to and restore from a backup, but never to modify or delete it. Think: ransomware.

## Approach
To tackle the various requirements above, my backup approach is broken down into a number of layers. Each serving a different purpose and with the ability to pick and choose how it is done and even switch tools on one layer, ideally without influencing other layers.

### Layer 1: Timestamped local copy
The computer storing the live version of my data shall have a local backup of these files. Meaning: after the first run of a backup, there will be two identical copies of that file. If the file is then changed and the backup runs again, three versions of that file are there: the live version, the (identical) new backup and and previous version as backup. Since the underlying storage is encrypted, no encryption is needed from the backup tool in this layer.

### Layer 2: Encrypted remote copy
As with any electronic (or even worse: mechanical) device, the question is never if it will fail but when. To counter that, I want my data to be copied onto a second device, like a USB harddrive, a NAS or a public cloud. This step will require encryption on file-level. Encrypting whole disks is nice, but not nicely compatible with cloud storage. But this layer not have to care about keeping a timeseries of the data, as that requirement was already covered in the previous layer.

### Layer 3: Automation & Verification
The two previous layers can be executed manually. And should be, in the first approach, to obtain a better understanding of the tools and the process. But for a permanent use, there should be no manual trigger necessary. Both for creating the backups as well as for the verification that a restore will produce the original data again.

## Implementation

### Local time series
For this task, I have chosen [rdiff-backup](https://github.com/rdiff-backup/rdiff-backup) - to quote from their project repository:

> rdiff-backup is a simple backup tool which can be used locally and remotely, on Linux and Windows, and even cross-platform between both. [..] Beside its ease of use, one of the main advantages of rdiff-backup is that it does use the same efficient protocol as rsync to transfer and store data. Because rdiff-backup only stores the differences from the previous backup to the next one (a so called reverse incremental backup), the latest backup is always a full backup, making it easiest and fastest to restore the most recent backups, combining the space advantages of incremental backups while keeping the speed advantages of full backups (at least for recent ones).

Install rdiff-backup according to the instructions for your platform.

Decide, whether you want to include all files and directories in the backup. I am excluding most of my dot-directories, as especially `.local` or `.cache` tend to grow big fast and for me contain no valuable content. This command will print those in a file (run in your home directory):

```
find . -maxdepth 1 -type d -name '.*' > .rdiff-exclude 
```

However, some directories like `.ssh` you want to backup! So review the `.rdiff-exclude` file and remove each line again when a directory should not be excluded from the backup.

Prepare the target folder where the backup will be stored. Change, if you are not happy with the location.
```
sudo mkdir -p /home/.backups/user1
sudo chown user1 /home/.backups/user1
```

Now it is time to create the first backup. We `cd` into the directory so that the relative paths in the `.rdiff-exclude` file work; otherwise those need to be absolut paths.
```
cd /home/user1
rdiff-backup --no-fsync --exclude-special-files --exclude-globbing-filelist .rdiff-exclude . /home/.backups/user1
```

Depending on the amount of files, this may take a while. However, running this command a second time is significantly faster. And subsequent runs will only take more disk space if you have modified any files. Under `/home/.backups/user1` you will now find your backup as plain files. No tool is needed to access the latest full backup.

### Encrypted Remote Copy
* File level encryption with `gocryptfs`
* Allows "reverse" mode where plaintext files on the disk are accessed via an overlay mountpoint and are seen as encrypted there
* Synchronization with external HDD or NAS via `rsync`
* Synchronization with public cloud (e.g. Dropbox) with their tools

Initialize gocryptfs in the backup folder and create a temporary mountpoint for the encrypted view:
```
gocryptfs -reverse -init /home/.backup/user1/
mkdir /tmp/user1
```

Unlock and monut the encrypted folder. After that you should see encrypted files in `/tmp/user1`
```
gocryptfs -reverse /home/.backup/user1/ /tmp/user1
```

Synchronize the encrypted files to the external storage medium:
```
rsync -av --delete /tmp/user1/ /path/to/my/usb-hdd/encrypted-backups-of-user1
```

On the external storage or the cloud service there should now be a copy of the encrypted backup. Subsequent runs of the synchronization should finish significantly faster, as only changed files will be transferred.

Feel free to have more than one remote copy of your local backup.

### Verification

### Automation

## Summary
Are all requirements met?
