
# Gerrit Backup

This is a backup tool like [duplicity](http://duplicity.nongnu.org/) but custom build to backup Gerrit servers, with the option for pre- and post- tasks: -

- Stopping / Starting Ubuntu Services.
- Stopping / Starting Jenkins jobs.
- Run MySQL commands, like dump.

## Install build requirements

> Ubuntu

~~~bash
sudo apt-get install -y build-essential libssl-dev libffi-dev python-dev
~~~

> Mac OS X (Xcode needed)

~~~bash
xcode-select --install

brew install python openssl libffi
~~~

## Install python requirements

~~~bash
pip install -r requirements.txt
~~~

## Remote Dependencies

~~~
curl --silent --show-error --retry 5 https://bootstrap.pypa.io/get-pip.py | sudo python2.7
sudo pip install boto
~~~

-- or --

~~~
sudo apt-get install -y python-pip
sudo pip install boto
~~~

### Notes: -

- Remote user need password less sudoer access.

## Lint python script

~~~
flake8 **/*.py
~~~

## Code Metrics

~~~
radon cc gerrit_backup_tool -a -nc

radon cc . -a -nc
~~~

## Generate Docs, with 'sphinx'

~~~
sphinx-build -b html . docs
~~~

## Configuration

This following section can be used in the configuration file: -

|Section/Values|Description|Example Value|
|---|---|---|
|**```[gerrit]```**|||
|```gerrit_path```||```/var/gerrit/review/```|
|```gerrit_username```||gerrit|
|```gerrit_group```||gerrit|
|**```[backup]```**|||
|```ssh_username```||vagrant|
|```ssh_key_file```||```~/.vagrant.d/insecure_private_key```|
|```ssh_hostname```||gerrit-server|
|```ssh_port```||22|
|**```[backup_api]```**|||
|```api_url```||http://gerrit-server|
|```api_verify_ssl```||False|
|**```[restore]```**|||
|```ssh_username```||vagrant|
|```ssh_key_file```||```~/.vagrant.d/insecure_private_key```|
|```ssh_hostname```||gerrit-server|
|```ssh_port```||22|
|**```[pre_tasks:stop_services]```**|||
|```services```||gerrit,apache2|
|```run_remotely```||True|
|```run_for_commands```||backup,restore|
|**```[pre_tasks:command:gerrit_reindex]```**|||
|```command```||```java -jar ./bin/gerrit.war reindex```|
|```pwd```||```/var/gerrit/review```|
|```run_remotely```||True|
|```run_for_commands```||backup|
|```sleep```||5|
|**```[pre_tasks:database_command:read_only_mode]```**|||
|```command```||```FLUSH TABLES WITH READ LOCK; SET GLOBAL read_only||ON;```|
|```run_remotely```||True|
|```run_for_commands```||backup|
|**```[pre_tasks:database_command:read_only_mode_check]```**|||
|```command```||```SELECT @@global.read_only;```|
|```run_remotely```||True|
|```run_for_commands```||backup|
|**```[post_tasks:database_command:read_only_mode]```**|||
|```command```||```SET GLOBAL read_only||OFF; UNLOCK TABLES;```|
|```run_remotely```||True|
|```run_for_commands```||backup,restore|
|**```[post_tasks:database_command:read_only_mode_check]```**|||
|```command```||```SELECT @@global.read_only;```|
|```run_remotely```||True|
|```run_for_commands```||backup,restore|
|**```[post_tasks:command:gerrit_reindex]```**|||
|```command```||```java -jar ./bin/gerrit.war reindex```|
|```pwd```||```/var/gerrit/review```|
|```run_remotely```||True|
|```run_for_commands```||restore|
|```sleep```||25|
|**```[post_tasks:start_services]```**|||
|```services```||gerrit,apache2|
|```run_remotely```||True|
|```run_for_commands```||backup,restore|
|```sleep```||5|
|**```[backup_structure]```**|||
|```database_folder```||database|
|```repos_folder```||repos|
|```repos_list_folder```||```repos_list```|
|**```[backup_folder]```**|||
|```backup_folder```||```/vagrant/gerrit_backup```|
|**```[script]```**|||
|```repo_list_filename```||```gerrit_backup_repos.txt```|
|```script_python_filename```||```gerrit_backup.py```|
|**```[database]```**|||
|```database_host```||localhost|
|```database_username```||gerrit|
|```database_password```||gerrit|
|```databases_name```||gerrit|
|```database_dump_file```||```gerrit_mysql_dump.sql```|

For an example configuration that works with [jpnewman\_ansible_gerrit](https://github.com/jpnewman/jpnewman_ansible_gerrit) look at file ```gerrit_backup.cfg``` within the same folder as this ```README.md```.

**NOTES: -**

- If ```ssh_key_file``` is not define the key should be added to ```ssh-agent```: -

~~~
ssh-add ~/.vagrant.d/insecure_private_key
~~~

## Supported Pre / Post Tasks

### ```stop_jenkins_job```

~~~
[post_tasks:start_jenkins_job:gerrit-replicate-git]
jenkins_url = http://jenkins-server/
run_remotely = False
run_for_commands=backup,restore
~~~

### ```start_jenkins_job```

~~~
[pre_tasks:stop_jenkins_job:gerrit-replicate-git]
jenkins_url = http://jenkins-server/
run_remotely = False
run_for_commands=backup,restore
~~~

### ```stop_services```

~~~
[pre_tasks:stop_services]
services = gerrit,apache2
run_remotely = True
run_for_commands=backup,restore
~~~

### ```start_services```

~~~
[post_tasks:start_services]
services = gerrit,apache2
run_remotely = True
run_for_commands=backup,restore
~~~

### ```command```

~~~
[pre_tasks:command:gerrit_reindex]
command = java -jar ./bin/gerrit.war reindex
pwd = /var/gerrit/review
run_remotely = True
run_for_commands=backup
sleep = 5
~~~

### ```database_command```

~~~
[pre_tasks:database_command:read_only_mode]
command = FLUSH TABLES WITH READ LOCK; SET GLOBAL read_only = ON;
run_remotely = True
run_for_commands=backup
~~~

## Run

### Get Repo List

~~~
python gerrit_backup_tool gerrit_backup.cfg --get-repo-list

python gerrit_backup_tool `PWD`/gerrit_backup.cfg --get-repo-list
~~~

### Backup

~~~
python gerrit_backup_tool gerrit_backup.cfg --backup 2>&1 | tee gerrit_backup.log

python gerrit_backup_tool gerrit_backup.cfg --backup --use-screen 2>&1 | tee gerrit_backup_screen.log
~~~

### Restore

~~~
python gerrit_backup_tool gerrit_backup.cfg --restore 2>&1 | tee gerrit_restore.log
~~~

### Disk Usage

~~~
python gerrit_backup_tool gerrit_backup.cfg --diskusage 2>&1 | tee gerrit_backup_diskusage.log
~~~

### Run pre-tasks only

~~~
python gerrit_backup_tool gerrit_backup.cfg --pre-tasks-only 2>&1 | tee gerrit_backup_pre_tasks_only.log
~~~

### Run post-tasks only

~~~
python gerrit_backup_tool gerrit_backup.cfg --post-tasks-only --repo-list gerrit_backup_repos.txt 2>&1 | tee gerrit_backup_post_tasks_only.log
~~~

## Dry-Run

~~~
python gerrit_backup_tool gerrit_backup.cfg --backup --dry-run --verbose 2>&1 | tee gerrit_backup_dry_run.log
~~~

> To view only ```INFO``` messages / progress you can tail the log and grep for ```INFO:```: -

~~~
tail -f gerrit_backup_dry_run.log | grep 'INFO:'
~~~

## Load previously generated repo list, dry-run

~~~
python gerrit_backup_tool gerrit_backup.cfg --backup --repo-list gerrit_backup_repos.txt --dry-run 2>&1 | tee gerrit_backup_dry_run.log
~~~

## Dry-Run python script on target box

~~~
cd gerrit_backup_tool

python gerrit_backup.py gerrit_backup.cfg --diskusage --dry-run

python gerrit_backup.py gerrit_backup.cfg --backup-database --dry-run
python gerrit_backup.py gerrit_backup.cfg --backup-repos --dry-run
~~~

## UnitTest

~~~
py.test --html=report.html
~~~

## Line Count

~~~
find . -name '*.py' | xargs wc -l
~~~

## Validate

~~~
python3 validate.py
~~~

## NOTES

- Backups are not atomic / transactional

## License

MIT / BSD

## Author Information

John Paul Newman
