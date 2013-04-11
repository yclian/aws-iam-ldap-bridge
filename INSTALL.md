TODO:
- manual configuration

Introduction
============

This project provides an implementation of Unix PAM authentication using AWS IAM ("Identity and Access Management").
It removes the need for everyone to login using shared certificates, or maintain copy of all certificates on all instances
for all the users of your team (and operations, and support etc.). It also simplifies the scripted authenticated access from one machine to another.

It works by integrating into ApacheDS LDAP server as a plugin. The plugin periodically populates the LDAP directory location with the
users and groups from AWS IAM. When Linux PAM is configured with LDAP authentication this allows for authenticating the Linux users against
these replicated AWS IAM users which effectively as if you authenticated against AWS IAM directly.

The authentication works based on using the AWS IAM user name as Linux account name, and (the first) Secret Key as password.
After login, the users will have the Linux groups corresponding to the IAM groups that were assigned to them. All accounts (users and groups) are
created and updated automatically. If the user does not have access key/secret key the account is not created in LDAP.

> *Note:* The user's AWS Secret Keys are never stored in any persistent storage or logs. Only user names and accessKeys are stored in LDAP, and
those are filtered out of search results if `ads-dspasswordhidden` property is set.

By default, the users are cached in LDAP at ou=users,dc=example,dc=com and groups as ou=groups,dc=example,dc=com. You can change
that by modifying the rootDN in auth.ldif and importing it back into the server.

> _You should also be aware that this project is in its early stages. No formal security assessment has been done against it. Considering that
you are NOT advised to use it for any security sensitive application. Feel free to evaluate this project but be aware that you use it at your own risk._

Quick start
===========

Download the binary package from <a></a>.
This package contains a self-contained installation of ApacheDS 2.0.0-M11 with AWS IAM bridge. It is designed to be used
straight away on any Linux system which has Java 6 without any manual configuration. For example, it can be embedded into
an AWS AMI and used for all your servers to allow the AWS IAM authentication of Linux users.

To start using the server, you need to configure your AWS access and secret keys that the authenticator is going to use
to fetch the users and groups, and authenticate with AWS IAM on their behalf.

1. Extract the contents of the archive

1. Edit the modify.ldif and change the values for `accessKey` and `secretKey`.

    The specified user must have the following permissions:

    * Read/Write access to DynamoDB (you can restrict read/write to specific DynamoDB tables with names `IAMUsers`, `IAMGroups`, `IAMRoles`)
    * Read access to IAM List* and Get* operations.

1. Start the ApacheDS server (assuming Linux):

        bash apacheds.sh &

        sleep 10

1. Apply the AWS configuration

        ldapmodify -H ldap://localhost:10389 -D uid=admin,ou=system -w secret -x -f modify.ldif

1. Restart the ApacheDS server

        killall apacheds

        bash apacheds.sh &

    After that the server should be filled with the users/groups. You can verify that by executing the following:

        ldapsearch -H ldap://localhost:10389 -D "uid=admin,ou=system" -x -w secret -b "dc=example,dc=com" "(objectclass=posixaccount)"

    You should get a list of your IAM accounts.

> *Note:* it is up to you to configure the PAM LDAP or similar authentication mechanism. You can use this guide for configuration <http://wiki.debian.org/LDAP/PAM/>.
Pick the `libnss-ldapd`/`libpam-ldapd` option as I found it to work the best with ApacheDS (on Ubuntu). You'll also need to modify /etc/ssh/sshd_config by
 commenting out the line of `#PasswordAuthentication no`.
After successful configuration of LDAP and NSLCD you should be able to see the users and groups using `getent passwd` and `getent group`
If that works, you should now be able to login using the username of one of your IAM accounts, and using the secret key as the password.

Security notes
==============

The default configuration is _INSECURE_ however you are free to alter it to your requirements. It should not affect the behavior of the custom authenticator.

You may want to change the following defaults:

- The default `uid=admin,ou=system` password, `secret` by default
- Disable anonymous binds
- Enable TLS/SASL or LDAPS
- Change the rootDN where the users/groups are stored
- Enable ACL and change the permissions for the `dn: ads-authenticatorid=awsiamauthenticator,ou=authenticators,ads-interceptorId=authenticationInterceptor,ou=interceptors,ads-directoryServiceId=default,ou=config`,
  as well as the rootDN.

    This entry stores the AWS Access Key and Secret Key for the authenticator, and rootDN stores all the information about accounts. You need to prevent any logged in users
    from modifying the account's keys and group information (only admin should be allowed to do that).

Configuring an existing ApacheDS LDAP server
============================================
If you installed the ApacheDS by other means, you can add the authenticator to it manually:
TBD

Assumptions
===========
- Users have only one access key. If you users have more than one access key, the authenticator will pick the first of them for authentication.