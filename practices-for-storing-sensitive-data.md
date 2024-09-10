# Practices for secrets handling

Written by Stefan, on 10 September 2024.
Tags: `opsec` `linux` `mac`

> More to come!

# 1. Introduction

If you're a developer or admin, you accumalate all sorts of files on your devices.
Some of them contain secrets that should not stay on your device.
Some of them contain keys to someone's kingdom.
Devices may break, get lost or stolen.  What happens now?
Over time, such files will pose an ever increasing risk if not handled well.
Let's mitigate the risks by understanding and managing our files better.

# 2. Grading secrets

By understanding our secrets we can label them and thread them based on their grade(s), for example secret-S0-G0 and so on. This way we avoid having to store the grade information externally.

Grade by sensitivity
* S0 secrets - grant access to NO sensitive information
* S1 secrets - grant access to sensitive information
* S2 secrets - grant access to a lot of sensitive information
* S3 secrets - grant access to a sensitive information and PII
* etc.

Grade by access protection
* **A0** secrets - are plain readable, maybe encoded
* A1 secrets - require 1 level of access validation
* A2 secrets - require 2 levels of access validation (2nd factor)
* A3 secrets - require 3 levels of access validation (3nd factor)
* A4 secrets - require 3 " " .. and location validation (3rd party validating geolocation, IP or similar)
* etc.

Note:

> The difference between A3/A4 does not concern us on our device
> directly but should be kept in mind. 
> 
>We don't distinguish between
> human-readable and encoded (for example, base64)
>
> Grading may require updating the labels if scope changes
> 

Lets take an ssh key as an example allowing access to a server hosting a public basic page. That's it. The ssh key is not password protected: `my-ssh-key-S0-A0.key`


# 3. A place for secrets
We gather all secret's locations all in one place.
A sort of meta folder that points to all locations or at least includes them.

All locations of secrets should be known. By knowing their locations we can:
* Avoid accidentally mishandling them
* Expire them by safely deleting them ('shredding' or 'wiping')
* Using our grading to distinguish and manage them better

# 4. Handling of secrets

A checklist for getting things right

 - [ ] Create a folder `secrets` in your home folder.
 - [ ] Set the permissions so only you can read and change it
 - [ ] Put all sensitive files files there you wish to expire at one point
 - [ ] Put links to other locations you into the secrets folder
 - [ ] Label files and links according to our grading system
 - [ ] Verify files and folders in place have correct permissions, if not: alert and fix
 - [ ] Using grading system to determine what to expire, when and how
 - [ ] Have a regular task do the expiration and validation / fix of permissions.
 - [ ] Verify regular task ran, using alerting or monitoring

**Warning**: 
Some files cannot be moved to `secrets` but can still by linked and labled
```
ln -s ~/.aws/config ~/secrets/dot-aws-config-S0-A0
ln -s ~/.aws/credentials ~/secrets/dot-aws-credentials-S3-G4
```


# 5. Handling AWS config

When using AWS on the command line, one option is to use `aws-cli`, the official CLI client. By default, it allows to add a 'profile' - a named config set to use AWS services.

Such a config is often bound to a username (AccessKey) and password (AccessKeySecret). The `aws-cli` stores the settings of a profile in `~/.aws/credentials` .Other config is located in `~/.aws/config`
Unfortunately `~/.aws/credentials` needs to be protected better as API calls would
not require another factor of authentication

Use [aws-vault](https://github.com/99designs/aws-vault/) to store secrets in a trustworthy system keychain requiring the 
keychain password to open it. Also avoids accidental pasting errors and polluting shell history.. 

So next time you need to run an AWS command, run:
`aws-vault exec my-aws-profile aws s3 ls`
.. and it will prompt you interactively for the keychain password. 

To use a MFA device in a shell, add your `mfa_serial=MY_MFA_ARN` to your `~/.aws/config` 
aws-vault will then prompt you for your MFA token:

```
aws-vault exec my-aws-profile -- aws s3 ls
Enter MFA code for arn:aws:iam::************:mfa/aws-dev:

```

**Note**: Some actions (IAM) ONLY work with an MFA device config present


# 6. GPG for encrypting and password manager

The GPG key unlocks all passwords in the password-store. 
The key can also be used to encrypt sensitive files of grade A0 to make them A2 - GPG key and password required.

Create a GPG key pair and protect it with a password. Use this key when doing the `pass init` step for the password-storage.

# 7.  `pass` for password-storage

Handy command line [program](https://www.passwordstore.org/) to store, generate and search passwords and other notes. Pretty basic but does the job.  The data resides in `~/.password-store`

The password-storage is encrypts 'folders' automatically using your GPG key.

It therefore makes all passwords stored A2 - GPG key and GPG key password required to open.

> Note: It will be linked to our `secrets` folder and synched remotely
> to a git repo. 

* Clone password-store from remote GitHub if locally not present or lost
* Search password with `pass find` and copy to the clipboard when needed
* Make sure to push local changes to the remote with `pass git push`

# 8. Handling MySQL connections

I'm using 'login-paths' (a config bundle basically) to connect to MySQL / MariaDB hosts which are stored in `~/.my.cnf`in binary form. 

> Note: Of course, secrets can also be plainly stored in the
> password-store. You may want to consider storing a
> connection string in the password-store instead. 
> When adding host, user and database it gets
> ugly in password-store quickly

Label the `~/.my.cnf` as A0, it's not encrypted.
Or encrypt / decrypt it on the demand with GPG and label it as  A2 instead.

For ease of use, create login profiles like so and use them
`mysql_config_editor set --login-path=my-mysql-1 --user=root --host=127.0.0.1 --password`

`mysql_config_editor print --all`
`mysql --login-path=my-mysql-1`



# 8.  SSH for transit

Transferring and backing up our `secrets` may be required to ensure it's data safety.
SSH protects sensitive files in transit.

* Create a ssh keypair with a password. The password acts as a 2nd factor.
* The private key in `~/.ssh/id_YOUR_ID`can be labeled A1, it has a password
* Store the password in password-store
* Create a `~/.ssh/config`to connect to hosts easily
* Use `ssh-agent` to load they key so you don't have to type the password on every connection with `ssh-add ~/.ssh/my-ssh-key.key`

> Note: Also consider the server-side SSH implementation settings

```
Host backup.*
  User john
  ForwardAgent yes
  IdentityFile ~/.ssh/id_john
```
