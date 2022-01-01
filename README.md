# Passbolt Tutorial
This project aims to make it much easier to deploy Passbolt through Docker and
the instructions below will guide you through setting up.


### Preqrequisites
A Linux box running Docker and [Docker-compose](https://blog.programster.org/debian-8-install-docker-compose).
I would recommend using Debian 11 ([corresponding install script](https://blog.programster.org/debian-11-install-docker)).
**The instructions below assume you are running Debian/Ubuntu.**

One really needs SSL certificates to use Passbolt. You can get these for free
through [Lets Encrypt](https://letsencrypt.org/). I would recommend generating
some beforehand through the use of DNS based resolution.

Passbolt really needs to be able to send emails and this is ideally
performed through the use of an SMTP server. Luckily its easy to do
this through a free gmail account, for which the example environment
file has been mostly filled in for you with.


## Steps
Clone this repository onto the Docker server that is going to act as your
Passbolt server.

You can do this with git:
```bash
git clone https://github.com/programster/tutorial-passbolt-docker.git
```

or plain wget:
```bash
wget -O https://github.com/programster/tutorial-passbolt-docker/archive/refs/heads/main.zip
unzip main.zip
```

### Create Environment File
Run the following command to create your own .env file from the 
example/template.

```bash
cp .env.example .env
```

Fill in as many details as you can. You should be able to fill in all the 
details except for `GPG_KEY_FINGERPRINT`, which will com up later in the
steps instructions.


### Generate GPG Key
We are going to generate a GPG key that will be used by our Passbolt server. 
This may or may not be performed on the Passbolt server itself. It can be
faster to generate these off-server if you just deployed a VPS and it doesn't
have much/any entropy yet.

```bash
gpg --full-generate-key
```

When prompted, enter the following answers:
- `(1) RSA and RSA (default)`
- `2048` for key length
- `0` for does not expire
- Enter your real name, or something generic like `support` or `passbolt-admin`
- Enter an email address for the key. E.g. `support@passbolt.mydomin.com`
- `Passbolt GPG key` for the comment.
- `O` to confirm that you are happy with the details.
- Make sure to **not** set a passphrase.

When that has been completed, you should see details of the generated key. This 
includes the keys fingerprint. Be sure to copy this fingerprint for later.

If you forget to make a note of the fingerprint, you can get it again by 
running:

```bash
gpg --list-keys
```
This will output something like:

```
3F4AF67909E17FBB164400D44247E2A9BC6DC166
```

Use this value to set the `GPG_KEY_FINGERPRINT` value in your `.env` file.


### Export The Key

```bash
gpg --armor --export-secret-keys $EMAIL > serverkey_private.asc
gpg --armor --export $EMAIL > serverkey.asc
```

Copy these key files to the server to within a folder at:
```
$HOME/passbolt/gpg/
```

...and change the permissions with:

```bash
sudo chown $USER:33 serverkey* && \
  sudo chmod 640 serverkey*
```

**Note:** The reason for setting group 33 is because this is the ID of the 
`www-data` user within the passbolt container. This user needs read access to 
the files in order to be able to work.


### Import GPG Key
Now we need to tell passbolt to import the GPG key that we injected through the
use of Docker volumes. We do this by running the following command:

```bash
docker exec passbolt \
  su -m --command "gpg --home /var/lib/passbolt/.gnupg --import /etc/passbolt/gpg/serverkey_private.asc" \
  -s /bin/sh www-data
```

You should get a message like the following:
```
gpg: key F0309B0CF553E5D6: "passbolt-admin (Passbolt GPG key) <support@mydomain.com>" not changed
gpg: Total number processed: 1
gpg:              unchanged: 1
```

After this,  you can change the permissions back on the GPG key files:

```bash
sudo chmod 640 serverkey*
```

### Start the Database
We need to "warm up" the database and give it a chance to set up the first time
we deploy. We can do this by deploying just the database service on its own 
with:

```bash
docker-compose up db
```

### Start Passbolt
After the database has finished setting up (you stop seeing lots of console 
output), then you can deploy the Passbolt service with:

```bash
docker-compose up app
```


### Create Initial User
We need to create our initial administrative user account that we will
be able to log into Passbolt with, and then use to send invites to other users.
This can be achieved by running:

```bash
EMAIL=myemail@gmail.com
FIRST_NAME=God
LAST_NAME=User
docker exec passbolt \
  su -m -c "bin/cake passbolt register_user -u $EMAIL -f $FIRST_NAME -l $LAST_NAME -r admin" \
  -s /bin/sh www-data
```

This should generate output similar to below:
```

     ____                  __          ____  
    / __ \____  _____ ____/ /_  ____  / / /_ 
   / /_/ / __ `/ ___/ ___/ __ \/ __ \/ / __/ 
  / ____/ /_/ (__  |__  ) /_/ / /_/ / / /    
 /_/    \__,_/____/____/_.___/\____/_/\__/   

 Open source password manager for teams
-------------------------------------------------------------------------------
User saved successfully.
To start registration follow the link provided in your mailbox or here: 
https://passbolt.mydomain.com/setup/install/95411c1b-13c6-4a51-9ec8-e1a66de7b45d/95411c24-a0b9-463b-bf60-3433d313d937
```

Go to the link in the output in order to complete your initial user's 
registration.


## Debugging

### Run Healthcheck
If you get an any issues with your addon not being recognized etc, be sure to 
perform a healthcheck first.

```bash
docker exec passbolt \
  su -m --command "./bin/cake passbolt healthcheck" \
  --shell /bin/sh www-data
```

### Enter Container
You can also enter the container and poke around by running:

```bash
docker exec -it passbolt /bin/bash
```


## Deployment
During the steps above, we deployed the database first and then the passbolt 
container. In future you can just deploy both silently with:

```bash
docker-compose up -d
```

## Updating Passbolt
When there are future updates of Passbolt, one can easily apply them 
by performing the following steps:

1. Backup your server, preferably through a VPS snapshot.
2. Go to the Dockerhub [Passbolt tags page](https://hub.docker.com/r/passbolt/passbolt/tags) and find the latest appropriate tag (e.g. 3.5.0).
3. Change the docker-compose.yml file to use this tag instead.
4. Run `docker-compose pull && docker-compose up -d`

The same goes for MariaDB with the 
[MariaDB tags page](https://hub.docker.com/_/mariadb?tab=tags).

**Warning:** I would not recommend upgrading major release versions without
checking online that this is even possible to do safely first.


## References
* [Dockerhub - passbolt/passbolt](https://hub.docker.com/r/passbolt/passbolt/)
* [Community.passbolt.com - Firefox 85.0.2 (64-Bit) - Add-on is not recognized](https://community.passbolt.com/t/firefox-85-0-2-64-bit-add-on-is-not-recognized/3849/3)
