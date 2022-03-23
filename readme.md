<img width="692" height="475" src=":/ab746a6e66694d5986f5696ba319f932"/>

We [started](https://raduzaharia.medium.com/managing-user-identities-in-a-network-703094e4835a) talking about managing identities in your local home network, the first use case for that being correcting the [access rights](https://raduzaharia.medium.com/thoughts-on-nfs-file-sharing-d1b6d3c22a9) for NFS shares. There are two main choices for an identity server in the Linux world: OpenLDAP and FreeIPA. OpenLDAP is like React framework for Javascript, it’s just a LDAP database with some access and management services, while FreeIPA is like Angular: a full framework, with everything included, as well as web tools for directory management.

## Server setup

<img width="692" height="389" src=":/a75ac6d2a3c04575b73da763c780dc29"/>

Photo by [Taylor Vick](https://unsplash.com/@tvick?utm_source=medium&utm_medium=referral) on [Unsplash](https://unsplash.com/?utm_source=medium&utm_medium=referral)

But for now let’s talk about installing OpenLDAP on Ubuntu Server on a Raspberry PI 4. The choice of hardware and operating system is not very relevant as the packages are available for all Linux distributions. OpenLDAP is also very lightweight, considering it’s just a database with some easy services so it will work on a Raspberry PI 3, maybe even 2. RAM consumption is low enough: I tested it on a 1 GB board and it was all good. So let’s install OpenLDAP on the server:

<a id="1905"></a>#apt install slapd ldap-utils
#dpkg-reconfigure slapd

When running `dpkg-reconfigure slapd` you will set the domain name and a LDAP password. We will consider the password to be `ldap-password`with the domain `yourdomain.com`. It will also ask if you want to make root the admin of the LDAP database. I recommend not to do so. It is better to have a dedicated user as the LDAP admin.

Next, change the domain name in `/etc/idmapd.conf` to your domain `(Domain = yourdomain.com`. The line is at the top of the file in the `[General]` section and it’s commented. Don’t forget to uncomment it. Now let’s change the LDAP admin password. You will need to create a file called `roof.ldif` which should contain:

<a id="3b52"></a>dn: olcDatabase={0}config,cn=config
changetype: modify
replace: olcRootPW
olcRootPW: admin-password

The file above is also on my [github](https://github.com/raduzaharia-medium/openldap/blob/main/root.ldif). You shoud replace `admin-password`with a strong password. Then you can run:

<a id="6a37"></a>#ldapmodify -Y EXTERNAL -H ldapi:/// -w "" -f root.ldif

Next we add the LDAP content. Specifically, the users and the groups. We will see in the next articles how to add more content: sudoers and automounts. For now, let’s create the `users.ldif`(available on [github](https://github.com/raduzaharia-medium/openldap/blob/main/users.ldif)):

<a id="d7f4"></a>version: 1<a id="19eb"></a>dn: ou=people,dc=yourdomain,dc=com
objectClass: organizationalUnit
ou: people<a id="5b0b"></a>dn: uid=user1,ou=people,dc=yourdomain,dc=com
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: shadowAccount
cn: User Name
gidNumber: 10000
homeDirectory: /home/ldap/user1
sn: Name
givenName: User
uid: user1
uidNumber: 10000
displayName: User Name
gecos: User Name
loginShell: /bin/bash
mail: user1@yourdomain.com
userPassword: encryptedPasswordStringHere

And `groups.ldif`(also on [github](https://github.com/raduzaharia-medium/openldap/blob/main/groups.ldif)):

<a id="5d0a"></a>version: 1<a id="4998"></a>dn: ou=groups,dc=yourdomain,dc=com
objectClass: organizationalUnit
ou: groups<a id="c27c"></a>dn: cn=ldap-users,ou=groups,dc=yourdomain,dc=com
objectClass: posixGroup
cn: ldap-users
gidNumber: 10000
memberUid: user1

If you look at `users.ldif` you will see we are adding a single user called `user1` and note the `dc=yourdomain,dc=com`. You should change all that to be the actual user name and the actual domain. The same goes for the home directory we are configuring at `/home/ldap/user1`. It should be the correct user name. Also not that we are putting the `LDAP` user homes in a `/ldap` folder. This is not necessary, it’s just how I configured mine. The `sn` entry in `users.ldif` stands for surname or family name and `givenname` is for first name. For `userPassword` you need to add a strong initial password which the user should change after logging in. The password should be a Salted SHA 1 string (SSHA) and will look like this: `{SSHA}DkMTwB;+a/3DQTxCYEApdUtNXG`.

Also note that we assign `user1` to group 10000 (the `gidNumber` entry in `users.ldif`). And we give `user1` the UID 10000. If you add more users, you will need to increase each UID with 1. The group will remain the same for all as they are all part of group 10000. Note the `memberUid` entry. You will have one `memberUid` for each user you add.

Now let’s add the files in the LDAP database:

<a id="44ee"></a>#ldapadd -x -D cn=admin,dc=yourdomain,dc=com -w ldap-password -f users.ldif
#ldapadd -x -D cn=admin,dc=yourdomain,dc=com -w ldap-password -f groups.ldif

Now that the users and groups have been added, the server configuration is done. You should restart `slapd` or better yet, reboot the Raspberry PI.

## Client setup

<img width="692" height="461" src=":/21f6acfe52054ede8c5fed2ff72b5d44"/>

Photo by [Marius Masalar](https://unsplash.com/@marius?utm_source=medium&utm_medium=referral) on [Unsplash](https://unsplash.com/?utm_source=medium&utm_medium=referral)

Now we neet to setup the clients to login via the Raspberry PI OpenLDAP server. This procedure has to be done on each device that connects to the home network. And as you will login to your Raspberry PI via SSH using the same users, the client setup has to be made on the Raspberry PI server too. It will be an identity server and a client at the same time.

First install `sssd` with LDAP support and the OpenLDAP tools:

<a id="ed7c"></a>#apt install libnss-ldap libpam-ldap ldap-utils sssd

Now configure PAM (which breafly is the login service in Linux, I won’t enter into details here) by editing `/etc/pam.d/common-session`. The file has many services added that work during login and we need to add one more:

<a id="73e6"></a>session optional pam_mkhomedir.so skel=/etc/skel umask=077

What the above service does is it creates the user home folder if it does not exist when logging in. It creates it based on the `/etc/skel` folder that is present in any Linux distribution by default. If you put stuff in that folder it will be copied to all user home folders at the first login when the user home folder is created.

You should also check `/etc/nsswitch.conf` to have `sss` added as a login option like this:

<a id="ff9e"></a>passwd:     files sss systemd
group:      files sss systemd
netgroup:   sss files
automount:  sss files
services:   sss files

The above instructs the login manager to also consult `sssd` during login. Fedora for example has it by default and Ubuntu would add it automatically but each distribution does its thing and you should check nevertheless. Next we need to create the `/etc/sssd/sssd.conf` file:

<a id="0c2e"></a>#sudo nano /etc/sssd/sssd.conf

And we will write this in it (also on [github](https://github.com/raduzaharia-medium/openldap/blob/main/sssd.conf)):

<a id="dd23"></a>\[domain/yourdomain.com\]
id_provider = ldap
autofs_provider = ldap
auth_provider = ldap
chpass_provider = ldap<a id="bdab"></a>ldap_uri = ldap://192.168.68.100/<a id="ff8a"></a>ldap\_search\_base = dc=yourdomain,dc=com
ldap\_id\_use\_start\_tls = false
ldap\_tls\_reqcert = allow
ldap\_auth\_disable\_tls\_never\_use\_in_production = true<a id="f2a5"></a>cache_credentials = true
enumerate = true<a id="61a1"></a>\[sssd\]
domains = yourdomain.com<a id="05dc"></a>\[pam\]
offline\_credentials\_expiration = 0<a id="32d6"></a>\[nss\]
homedir_substring = /home/ldap

The above configuration should contain the proper domain instead of `yourdomain.com` and the same for `dc=yourdomain,dc=com`. Also `ldap_uri` should point to your Raspberry PI server so you should change it to the correct IP address. Also note the `homedir_substring`: if you configured your LDAP home directories to be in the `/ldap` folder as we did in the server configuration section, remember to do the same here with `/home/ldap`. If you left it `/home`, update here to `/home` too.

The option `ldap_auth_disable_tls_never_use_in_production` is the one that makes your setup work without encryption. I used OpenLDAP without encryption as it is a [private and secure](https://raduzaharia.medium.com/practical-security-for-the-home-network-5732d5ae1b86) home network. Setting up encryption is a bit of a hassle and you have to keep your encryption certificates up to date so I don’t necessarily recommend it. Again, this being a private home network with no outside access. For other environments you should setup encryption.

Ok, after creating `sssd.conf`, you need to update the permissions for it:

<a id="aaa4"></a>#sudo chmod 600 /etc/sssd/sssd.conf

If you don’t update the permissions `sssd` will not work. The client setup is done. You need to enable `sssd`:

<a id="ace5"></a>#sudo systemctl restart sssd
#sudo systemctl enable sssd

And you should probably reboot and login with `user1` or with the user you configured in the LDAP database.

## Further options

<img width="692" height="463" src=":/68a3218b91304b5284cbf3dabd48caa7"/>

Photo by [Jens Lelie](https://unsplash.com/@madebyjens?utm_source=medium&utm_medium=referral) on [Unsplash](https://unsplash.com/?utm_source=medium&utm_medium=referral)

The main login should work and you should be able to see all LDAP users at the login screen due to the `enumerate = true` option in `sssd.conf`. But of course you may be interested in setting up sudo rights for your users. And we will talk about just that in the [next article](https://raduzaharia.medium.com/adding-sudoers-to-openldap-e0a6b0c4c7ab), and then you might be interested in using [automount](https://raduzaharia.medium.com/adding-automount-to-openldap-a9dd1356864a) to automatically mount your Raspberry PI NFS shares when the users log in. We will talk about that in another article.

For now though, you did it. You configured OpenLDAP and your home network has a central database for its users. I hope you had fun doing this and I hope you learned a bit more about LDAP in general. It’s not such a big deal and OpenLDAP is very light, making the whole process very easy. We will see in a future article how [FreeIPA is configured](https://raduzaharia.medium.com/building-an-identity-server-with-freeipa-and-a-raspberry-pi-4-735d108db1fe), which is a very large jump from the easy and light setup we made here. Until then, stick around and don’t be shy to ask questions in the comment area. See you next time!