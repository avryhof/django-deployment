# Initial Git configuration

When you first setup git on your server, you want to be able to manage it as a user, but still remain secure.  This configuration will allow you to cache your credentials while using sudo, so your project folders aren't owned by a specific user.

First, become root

```sudo su```

Next, setup your credential helper

```git config --global credential.helper 'cache --timeout=43200'```

Next, return to being your own user

```exit```

Then set your identity

```
git config --global user.name 'Your Name'
git config --global user.email your@email.com
```
