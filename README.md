## Ansi-strano - A Capistrano like deployment process using Ansible

Before standardizing on Ansible for our automation framework, we used Capistrano (actually Caphub) for designing
our application deployment processes.  We really liked the way that Capistrano handled deployments by using symlinks
for configuration files and shared folders.  We also liked the ease of which we could roll back releases.  However,
it did have some drawbacks that always seeemed to cause us complications, namely the various release stages (dev, stage, prod)
and maintaining inventory lists.

Once we started using Ansible, it wasn't long before we said "Hey, wouldn't it be nice to have Capistrano's deployment
features available along side our Ansible inventory".  This tool is just that.  Its a basic deployment playbook meant for 
deploying a PHP web application (or other language) that may have a few configuration files.  Specifically in our example case
we're using it to deploy a CodeIgniter application.

Obviously, this isn't nearly a fully featured tool.  The goal here was to provide a base Ansible playbook that provided a similar deployment structure to Capistrano for a simiple use case.  As an Ansible user, you can easily take this as a starting point and add more complex functionality to meet your needs.

### Requirements
* First and foremost, you need Ansible installed and configured for your environment.  If you already have inventory setup or being pulled from a dynamic cloud script you can easily drop this in and configure it to use what you have.
* The user that you're using to run ansible across the hosts has sudo access.
* A github repo containing the application to be deployed

### Installation & Configuration
* Clone/download this repo
* Make adjustments to the options in group_vars/all.  The following options are available
 * hosts - This is the ansible option for which hosts should be targetted for deployment.  There are various ways this can be configured (static inventory, dynamic inventory & groups, etc).
 * repo_url - This needs to be set to the Github repo that contains the application being deployed
 * keep_releases - This is the number of past releases to keep available for rollback in the deployment structure on each system being deployed.
 * deploy_base - This is the base directory where the application will be deployed to.  Within this directory will be a directory structure similar to how Capistrano performs deployments with a symlink to the "current" active release.
 * shared_path - This is the base directory where things like log directories and config files will be placed and symlinked to.  You probably don't need to change this.
 * current_path - This is the name of the "current" active release symlink.  You probably don't need to change this.
 * repo_path - This is the directory where the actual repo is cloned/updated.  You probably don't need to change this.
 * file_owner - This is the user that will be set to own the deployed files.
 * file_group - This is the group that will be set to own the deployed files.
 * restart_services - This is a list of services to be restarted at the end of the deployment.  Leave this blank for none.
 * config_files - This is a hash of configuration files to be "templated" as part of the deployment process.  In this example, we're deploying a CodeIgniter application and using database.php and config.php.  The variables that are used to populate the various options in the files are set in the standard Ansible fashion in group_vars/all, though in a real world scenario these would most likely be set differently in different group_vars files based on environment, etc.
 ```
config_files:
  - { src: "templates/database.php", dest: "{{ shared_path }}/database.php" }
  - { src: "templates/config.php", dest: "{{ shared_path }}/config.php" }
 ```
 * symlinks - This is a hash of files/directories to be symlinked as part of the release process.  In most cases the sources will exist in the "shared" directory.  This is so that things like log directories can stay consistent across releases, and also so that the configration files can live outside of the release directory.  In this example we're symlinking our config files from the previous step as well as the logs and cache directories.
 ```
symlinks:
  - { src: "{{ shared_path }}/database.php", dest: "{{ release_path }}/application/config/database.php", create_src: False}
  - { src: "{{ shared_path }}/config.php", dest: "{{ release_path }}/application/config/config.php", create_src: False}
  - { src: "{{ shared_path }}/logs", dest: "{{ release_path }}/application/logs", create_src: True}
  - { src: "{{ shared_path }}/cache", dest: "{{ release_path }}/application/cache", create_src: True}
 ```

### Usage
* Basic usage assuming your hosts option is set to include all desired hosts:
```
ansible-playbook ansi-strano-deploy.yml

PLAY [Ansi-strano-deploy - Capistrano like deployment process using Ansible] ***

TASK: [Generate release timestamp] ********************************************
changed: [localhost -> 127.0.0.1]

TASK: [set_fact release_path='{{ deploy_base }}/releases/{{ timestamp.stdout }}'] ***
ok: [localhost]

TASK: [set_fact branch=master] ************************************************
ok: [localhost]

TASK: [Ensure our release related directories exist] **************************
ok: [localhost] => (item=/var/www/kiss-ci-base)
changed: [localhost] => (item=/var/www/kiss-ci-base/releases/20160625224414)
ok: [localhost] => (item=/var/www/kiss-ci-base/shared)
ok: [localhost] => (item=/var/www/kiss-ci-base/repo)

TASK: [Update source git repo] ************************************************
ok: [localhost]

TASK: [Copy the cached git copy] **********************************************
changed: [localhost]

TASK: [git checkout our desired branch] ***************************************
changed: [localhost]

TASK: [Update our config files if desired] ************************************
ok: [localhost] => (item={'dest': u'/var/www/kiss-ci-base/shared/database.php', 'src': 'templates/database.php'})
ok: [localhost] => (item={'dest': u'/var/www/kiss-ci-base/shared/config.php', 'src': 'templates/config.php'})

TASK: [Ensure our various symlink sources exist] ******************************
skipping: [localhost] => (item={'dest': u'/var/www/kiss-ci-base/releases/20160625224414/application/config/database.php', 'src': u'/var/www/kiss-ci-base/shared/database.php', 'create_src': False})
skipping: [localhost] => (item={'dest': u'/var/www/kiss-ci-base/releases/20160625224414/application/config/config.php', 'src': u'/var/www/kiss-ci-base/shared/config.php', 'create_src': False})
ok: [localhost] => (item={'dest': u'/var/www/kiss-ci-base/releases/20160625224414/application/logs', 'src': u'/var/www/kiss-ci-base/shared/logs', 'create_src': True})
ok: [localhost] => (item={'dest': u'/var/www/kiss-ci-base/releases/20160625224414/application/cache', 'src': u'/var/www/kiss-ci-base/shared/cache', 'create_src': True})

TASK: [Create our symlinks] ***************************************************
changed: [localhost] => (item={'dest': u'/var/www/kiss-ci-base/releases/20160625224414/application/config/database.php', 'src': u'/var/www/kiss-ci-base/shared/database.php', 'create_src': False})
changed: [localhost] => (item={'dest': u'/var/www/kiss-ci-base/releases/20160625224414/application/config/config.php', 'src': u'/var/www/kiss-ci-base/shared/config.php', 'create_src': False})
changed: [localhost] => (item={'dest': u'/var/www/kiss-ci-base/releases/20160625224414/application/logs', 'src': u'/var/www/kiss-ci-base/shared/logs', 'create_src': True})
changed: [localhost] => (item={'dest': u'/var/www/kiss-ci-base/releases/20160625224414/application/cache', 'src': u'/var/www/kiss-ci-base/shared/cache', 'create_src': True})

TASK: [Activate the release] **************************************************
changed: [localhost]

TASK: [Restart services as configured] ****************************************
changed: [localhost] => (item=apache24)

TASK: [Cleanup old releases] **************************************************
changed: [localhost]

PLAY RECAP ********************************************************************
localhost                  : ok=13   changed=8    unreachable=0    failed=0
```

* Or lets say the playbook hosts is set to all of your application servers, and you only want to deploy to those in the "production" group:
```
ansible-playbook ansi-strano-deploy.yml --limit=production
```

* Ooops, lets roll back to the release dated "20160625212658".  When running this task you will be prompted for the release to rollback to.
```
ansible-playbook ansi-strano-rollback.yml
Please enter the revision name to revert to.  This is the dated folder in the releases directory. [x]: 20160625212658

PLAY [Ansi-strano-rollback - Capistrano like rollback process using Ansible] ***

TASK: [Validate the release name first] ***************************************
ok: [localhost]

TASK: [Revert the symlinks to the specified version] **************************
changed: [localhost]

TASK: [Restart services as configured] ****************************************
changed: [localhost] => (item=apache24)

PLAY RECAP ********************************************************************
localhost                  : ok=3    changed=2    unreachable=0    failed=0
```