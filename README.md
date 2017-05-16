# Cronjob Manager plugin for Craft CMS

Craft plugin to programmatically manage GNU/Linux cronjobs.

## Features:
- Deal with your cronjobs in Craft.
- Create new cronjobs.
- Update existing cronjobs.
- Manage cronjobs of others users than runtime user using some sudo tricks (see below).

## Requirements:
- `crontab` command-line utility (should be already installed in your distribution).
- `sudo`, if you want to manage crontab of another user than runtime user without running into right issues (see below)

## Installation:
The library can be installed using Composer.
```   
composer require boboldehampsink/cronjob
```

## Usage:
The plugin is composed of three models:

- `CronjobModel` is an entity model which represent a cronjob.
- `Cronjob_RepositoryModel` is used to persist/retrieve your jobs.
- `Cronjob_AdapterModel` abstracts raw crontab read/write.

### Instanciate the repository:
In order to work, the `Cronjob_RepositoryModel` needs an instance of `Cronjob_AdapterModel` which handles raw read/write against the crontab.

```php
$crontabRepository = new Cronjob_RepositoryModel(new Cronjob_AdapterModel());
```

### Create a new cronjob and persist it into the crontab:
Suppose you want to create an new job which consist of launching the command "df >> /tmp/df.log" every day at 23:30. You can do it in two ways.

- In pure OO way:
```php
$cronjob = new CronjobModel();
$cronjob->minutes = '30';
$cronjob->hours = '23';
$cronjob->dayOfMonth = '*';
$cronjob->months = '*';
$cronjob->dayOfWeek = '*';
$cronjob->taskCommandLine = 'df >> /tmp/df.log';
$cronjob->comments = 'Logging disk usage'; // Comments are persisted in the crontab
```

- From raw cron syntax string using a factory method:  
```php
$cronjob = CronjobModel::createFromCrontabLine('30 23 * * * df >> /tmp/df.log');
```

You can now add your new cronjob in the crontab repository and persist all changes to the crontab.
```php
$crontabAdapter = new Cronjob_AdapterModel();
$crontabRepository = new Cronjob_RepositoryModel($crontabAdapter);
$crontabRepository->addJob($cronjob);
$crontabRepository->persist();
```

### Find a specific cronjob from the crontab repository and update it:
Suppose we want to modify the hour of an already existing cronjob. Finding existings jobs is made using some regular expressions. Search in made against the entire crontab line.
```php
$results = $crontabRepository->findJobByRegex('/Logging\ disk\ usage/');
$cronjob = $results[0];
$cronjob->hours = '21';
$crontabRepository->persist();
```

### Remove a cronjob from the crontab:
You can remove a job like this :
```php
$results = $crontabRepository->findJobByRegex('/Logging\ disk\ usage/');
$cronjob = $results[0];
$crontabRepository->removeJob($cronjob);
$crontabRepository->persist();
```
Note: Since cronjobs are internally matched by reference, they must be previously obtained from the repository or previously added.

### Work with the crontab of another user than runtime user:
This feature allow you to manage the crontab of another user than the user who launched the runtime. This can be useful when the runtime user is `www-data` but the owner of the crontab you want to edit is your own linux user account.

To do this, simply pass the username of the crontab owner as parameter of the `Cronjob_AdapterModel` constructor. Suppose you are `www-data` and you want to edit the crontab of user `bobby`:
```php
$crontabAdapter = new Cronjob_AdapterModel('bobby');
$crontabRepository = new Cronjob_RepositoryModel($crontabAdapter);
```

Using this way you will propably run into user rights issue.
This can be resolved by editing your sudoers file using 'visudo'.     
If you want to allow user `www-data` to edit the crontab of user `bobby`, add this line:
```
www-data        ALL=(bobby) NOPASSWD: /usr/bin/crontab
```
which tells sudo to not ask for password when you call `crontab` of user `bobby`

Now, you can have access to the crontab of user `bobby` like this :
```php
$crontabAdapter = new Cronjob_AdapterModel('bobby', true);
$crontabRepository = new Cronjob_RepositoryModel($crontabAdapter);
```
Note the second parameter `true` of the `Cronjob_AdapterModel` constructor call. This boolean tell the `Cronjob_AdapterModel` to use 'sudo' internally to read/write the crontab.   

### Enable or disable a cronjob
You can enable or disable your cron jobs by setting the "enabled" attribute of a `CronjobModel` object accordingly:
```php
$crontabJob->enabled = false;
```
This will have the effect to prepend your cronjob by a "#" in your crontab when persisting it.

## Todo
 - Create ElementType to manage cronjobs from the CP

## Changelog

### 0.1.1
 - Updated dependencies

### 0.1.0
 - Initial release that allows you to manage cronjobs programmatically

## Credits
Based on [TiBeN/CrontabManager](https://github.com/TiBeN/CrontabManager)
