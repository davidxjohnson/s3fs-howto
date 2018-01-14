# s3fs-howto
### Cheat-sheet on how to mount a S3 bucket to a Linux host (securely)

#### What's covered in this how-to: ####

* Use of S3FS fuze-based file system.
* Use of AWS CLI to create an S3 bucket, access-user and permissions.
* Creating a mount and enable it on start-up.
* Sync local data to the S3 mount (as a backup)

#### Why? ####

In the process of creating a Plex media center on a Raspberry Pi, I really wanted to have a way to periodically backup the local media store to the cloud. A S3 bucket seems ideal for a backup because it's inexpensive, secure, highly available and very durable. Wouldn't it be great if you could simply mount an S3 bucket to the media server and rsync to it? (Well ... maybe.)

#### AWS Security setup: ####
* First, if you don't already have an AWS account, [go ahead and get one](https://aws.amazon.com/s3/) (it's free).
* Once you have an account, don't login with the root account all the time. Be sure to [follow the security recommendations](https://console.aws.amazon.com/iam/home#/home) as a minimum:
  * setup multi-factor authentication for the root account
  * setup a separate [admin account](https://console.aws.amazon.com/iam/home#/users) for ... well ... regular administration.
  * create an [admin group](https://console.aws.amazon.com/iam/home#/groups) and attach the AdministratorAccess policy to it, then make the admin user a member of the admin group.
* Once you have an admin account, login as admin and [generate an access key](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_access-keys.html?icmpid=docs_iam_console). Download the access key and secret for safe keeping (we'll be using the access key-pair later in this how-to).

#### Install some software: ####

* Install the [S3FS fuze-based file system](https://github.com/s3fs-fuse/s3fs-fuse).
* Install the [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/installing.html).

#### Use the AWS CLI to configure your S3 bucket, user and access policy ####

Let's store your admin credentials for use in the CLI (use the AccessKeyId/SecretAccessKey pair that you generated previously for the admin user).

```bash
developer@vbox: aws configure --profile=admin

AWS Access Key ID [None]: AKIAI7IDCKIAGKSLMCQ
AWS Secret Access Key [None]: r+Mc7ucp9YzqOyoeYxtNKb/8RQqOImBoAggJAABk
Default region name [None]: us-east-1
Default output format [None]:
```
Create the S3 bucket using the admin account (substitute S3 bucket name and region for your particular use case):

```json
developer@vbox: aws --profile=admin s3api create-bucket --bucket dxj.media-server --region us-east-1

{
    "Location": "/dxj.media-server"
}
```
Create a user account that will be used to access the S3 bucket (substitute whatever name you'd rather use):
```json
developer@vbox: aws --profile=admin iam create-user --user-name=media-server-user
{
    "User": {
        "UserName": "media-server-user",
        "Path": "/",
        "CreateDate": "2018-01-13T15:53:12.711Z",
        "UserId": "AIDAIYUJKOUDY6RWCRZWM",
        "Arn": "arn:aws:iam::836808583228:user/media-server-user"
    }
}
```

Make note of the user Arn and bucket name from the above steps ... you'll need them for the next step. Create a bucket policy to grant the minimum access needed to mount a S3 bucket to you Linux host.

```json
developer@vbox: aws --profile=admin s3api put-bucket-policy --bucket=dxj.media-server --policy \
'{
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::836808583228:user/media-server-user"
            },
            "Action": [
                "s3:GetBucketLocation",
                "s3:ListBucket"
            ],
            "Resource": "arn:aws:s3:::dxj.media-server",
            "Condition": {}
        },
        {
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::836808583228:user/media-server-user"
            },
            "Action": [
                "s3:GetObject",
                "s3:DeleteObject",
                "s3:PutObject"
            ],
            "Resource": "arn:aws:s3:::dxj.media-server/*",
            "Condition": {}
        }
    ]
}'
```
**Note:** The above policy was generated using the [Policy Generator](http://awspolicygen.s3.amazonaws.com/policygen.html).

Next, generate an access key for your S3 bucket user:

```json
developer@vbox: aws --profile=admin iam create-access-key --user-name media-server-user
{
    "AccessKey": {
        "UserName": "media-server-user",
        "Status": "Active",
        "CreateDate": "2018-01-13T16:03:09.094Z",
        "SecretAccessKey": "87NNCA2GUoP6HWDZxyPypVHN156HWNCA2+bBi2LH",
        "AccessKeyId": "AKIAJ22SJDBHJ22DBHJQ"
    }
}
```

Finally, save the above user AccessKeyId/SecretAccessKey pair for use by the AWS CLI and S3FS:

```bash
# save credentials for the S3FS file system
# syntax is <S3 bucket name>:<key>:<secret>
developer@vbox: echo 'dxj.media-server:AKIAJ22SJDBHJ22DBHJQ:87NNCA2GUoP6HWDZxyPypVHN156HWNCA2+bBi2LH' >> ~/.passwd-s3fs
chmod 600 ~/.passwd-s3fs

# save the credentials for the AWS CLI
developer@vbox: aws configure --profile=media-server-user
```

#### Mount the S3 bucket using the S3FS fuze-based file system: ####

For testing purposes, let's mount the S3 bucket to the current user's home directory:

```bash
# make a directory for the drive mount
developer@vbox: mkdir -p ~/s3-drive/media-center

# allow the mount command to change ownership of files
developer@vbox: sudo sed -i s/\#user_allow_other/user_allow_other/g /etc/fuse.conf

# mount the S3 bucket with default permissions of 750 and owned by 1000:1000
developer@vbox: s3fs -o allow_other,uid=1000,gid=1000,umask=027  dxj.media-server ~/s3-drive/media-server

# is it mounted?
developer@vbox: mount | grep s3fs

s3fs on /home/developer/s3-drive/media-server type fuse.s3fs (rw,nosuid,nodev,relatime,user_id=1000,group_id=1000)

# copy files to the drive
developer@vbox: cd ~
developer@vbox:~$ rsync -rz media-server s3-drive
developer@vbox:~$ tree s3-drive/
s3-drive/
└── media-server
    ├── music
    │   ├── 1X28UjzT.mp3
    │   ├── _211rqv-.mp3
    │   ├── 3m5BvQ-A.mp3
    │   ├── cx-5Hwz0.mp3
    │   ├── _IiIHYiX.mp3
    │   └── WYhWs25z.mp3
    ├── pictures
    │   ├── hcTYFi_q.jpeg
    │   ├── KJmJzwYt.jpeg
    │   ├── Sz22kOxB.jpeg
    │   └── tL2JW2rc.jpeg
    └── videos
        ├── 8Zjty7f3.mpeg
        ├── AmmkZ9r2.mpeg
        └── tJiMVwOz.mpeg
```
Check the [S3 console](https://s3.console.aws.amazon.com/s3) to verify that the files arrived in the cloud.

While there, create a new directory using the S3 console. Check the drive mount in your Linux host to see if it shows there. It should be instantaneous.

**Note:** The umask, gid and uid options used with the mount command are a work-around where files and folders are created in the S3 bucket using other tools or the S3 console. Without these options, the files wouldn't be accessible by the current user.

#### Configuring the drive to mount on start-up
```bash
# move our experiment to a little more permanent place
developer@vbox: cd ~
developer@vbox: sudo umount /home/developer/s3-drive/media-server
developer@vbox: sudo mkdir -p /data/s3drive/
developer@vbox: sudo mv /home/developer/s3-drive/media-server /data/s3drive/media-server-backup
developer@vbox: sudo chown root:root /data/s3drive/media-server-backup
developer@vbox: sudo mv ~/.passwd-s3fs /etc/passwd-s3fs
developer@vbox: sudo chown root:root /etc/passwd-s3fs

# let's mount it and test it
developer@vbox: sudo s3fs  -o allow_other,uid=1000,gid=1000,umask=027  dxj.media-server /data/s3drive/media-server-backup
developer@vbox: rsync -rz --delete ./media-server/* /data/s3drive/media-server-backup/*

# mount on start-up
developer@vbox: echo 'dxj.media-server /data/s3drive/media-server-backup fuse.s3fs _netdev,allow_other,uid=1000,gid=1000,umask=027 0 0' | sudo tee --append /etc/fstab
```
Reboot your Linux host to test that the mount persists.

#### An alternative to rsync: ####

The following command bypasses the fuze file system and updates the S3 bucket direct:

```bash
# two-way sync
developer@vbox: aws --profile=media-server-user s3 sync /home/developer/media-server s3://dxj.media-server --delete
developer@vbox: aws --profile=media-server-user s3 sync s3://dxj.media-server /home/developer/media-server --delete

# known bug where s3 sync command does not remove empty folders.
developer@vbox: find /home/developer/media-server -type d -empty | xargs rm -rf
developer@vbox: find /data/s3drive/media-server-backup -type d -empty | xargs rm -rf
```

Which begs the question: If all you want is a backup of your media drive, use the AWS CLI instead of a drive mount? :)
