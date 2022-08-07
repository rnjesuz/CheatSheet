# pg_restore script
Run in a screen:
``` Shell
screen -S restore
```
Para sair do screen:

> ctrl + a + d

Para retornar ao screen e verificar estado do processo:
``` Shell
screen -r restore
```
Script:
``` Shell
#!/usr/bin/env bash
START_TIME=$(date)
START_DIFF=$(date +%s)
echo "Starting restore at $START_TIME"
psql --host=localhost --port=5432 --user=postgres --dbname=<db_name> -c '
    CREATE EXTENSION pg_repack;
    ALTER DEFAULT PRIVILEGES IN SCHEMA repack GRANT INSERT ON TABLES TO PUBLIC;
    ALTER DEFAULT PRIVILEGES IN SCHEMA repack GRANT USAGE, SELECT ON SEQUENCES TO PUBLIC;'
pg_restore -h localhost --port=5432 -U postgres -d <db_name> --role=postgres --format=directory --no-owner --no-privileges --no-acl --verbose <dump_diretory> 2>&1 | tee log-pgrestore.txt
#pg_restore -v -W -h localhost -d <dn_name> -C -Fc <dump_file> -c -U postgres 2>&1 | tee log-pgrestore.txt
END_TIME=$(date)
END_DIFF=$(date +%s)
echo "Ended restore at $END_TIME"
DIFF=$((END_DIFF - START_DIFF))
PRETTY_DIFF=$((DIFF/60))
echo "Restore took $PRETTY_DIFF minutes"
```

---
# Tirar thread stack dump
``` Shell
$ ansible-playbook ../playbooks/extract_jstacks.yml \
    --extra-vars '{"host":"services","interval":"5","iterations":"5","container_filter":"label=com.docker.compose.service=server"}' \
    --ask-vault-pass
```

``` Yaml
- name: Run jstack extractor script
  become: yes
  strategy: free
  hosts: "{{ host }}"
  tasks:

    - name: verifies if the bin directory exists in the remote machine
      stat:
        path: "/opt/<stub>/bin"
      register: bin_dir_stat_result

    - name: creates bin directory in remote machine if it does not exist
      file:
        path: "/opt/<stub>/bin"
        state: directory
      when: bin_dir_stat_result.stat.exists == False

    - name: copy jstack extractor script to remote machine if it doesn't exist
      copy:
        src: "../../tools/jstack-tools/jstack_extractor.sh"
        dest: "/opt/<stub>/bin/jstack_extractor.sh"
        mode: "0755"

    - name: execute jstack extractor script
      shell: "/opt/<stub>/bin/jstack_extractor.sh -t {{ interval }} -i {{ iterations }} -f {{ container_filter }}"

    - name: get remote jstack files
      synchronize:
        mode: pull
        src: "/tmp/jstacks/"
        dest: "~/Desktop/"

    - name: clean up remote jstack files
      file:
        state: absent
        path: "{{ item }}"
      with_items:
        - /tmp/jstacks
```
``` Yaml
- name: Run jstack extractor script
  become: yes
  strategy: free
  hosts: "{{ host }}"
  tasks:

    - name: verifies if the bin directory exists in the remote machine
      stat:
        path: "/opt/<stub>/bin"
      register: bin_dir_stat_result

    - name: creates bin directory in remote machine if it does not exist
      file:
        path: "/opt/<stub>/bin"
        state: directory
      when: bin_dir_stat_result.stat.exists == False

    - name: copy jstack extractor script to remote machine if it doesn't exist
      copy:
        src: "../../tools/jstack-tools/jstack_extractor.sh"
        dest: "/opt/<stub>/bin/jstack_extractor.sh"
        mode: "0755"

    - name: execute jstack extractor script
      shell: "/opt/<stub>/bin/jstack_extractor.sh -t {{ interval }} -i {{ iterations }} -f {{ container_filter }}"

    - name: get remote jstack files
      synchronize:
        mode: pull
        src: "/tmp/jstacks/"
        dest: "~/Desktop/"

    - name: clean up remote jstack files
      file:
        state: absent
        path: "{{ item }}"
      with_items:
        - /tmp/jstacks
```
``` Shell
#!/usr/bin/env bash

#####################################################################
# Script to take jstacks from the docker containers                 #
# Version: 2.0                                                      #
#####################################################################

set -euo pipefail

function iterate() {
	iterations=$1
	sleep_time=$2
	date=$3
	container_id=$4
	host=$(hostname)

	for iter in $(seq 1 $iterations)
	do

		echo "Call Number: $iter"
		output_file=/tmp/jstacks/${date}_${host}/jstack_${date}_${host}_${container_id}_${iter}.out
		echo "docker exec $container_id sh -c 'unset JAVA_TOOL_OPTIONS; jstack -l -e 1' > $output_file"
		docker exec $container_id sh -c 'unset JAVA_TOOL_OPTIONS; jstack -l -e 1' > $output_file
		sleep $sleep_time
	done
}

function setup() {
	DIRECTORY=/tmp/jstacks
	host=$(hostname)

	if [ ! -d "$DIRECTORY" ];
	then
		mkdir $DIRECTORY
	fi
	mkdir $DIRECTORY/$1_${host}
}

function options() {
    echo "Options"
    echo "-t | --time --> Time between jstacks"
    echo "-i | --iterations --> Number of jstacks to be taken"
    echo "-f | --filter --> Container filter (see https://docs.docker.com/engine/reference/commandline/ps/#filtering for more information)"
}

while [[ $# -gt 0 ]]
do
	key="$1"

	case $key in
		-t|--time)
		TIME="$2"
		shift
		shift
		;;
		-i|--iterations)
		ITERATIONS="$2"
		shift
		shift
		;;
		-f|--filter)
		CONTAINER_FILTER="$2"
		shift
		shift
		;;
		-h|--help*)
		shift
		options
		exit
		;;
		*)
		echo "Use --help for further information"
		exit
		;;
	esac
done

current_date=$(date "+%Y-%m-%d_%H-%M-%S")

setup $current_date

for CONTAINER_ID in $(docker ps -q --filter $CONTAINER_FILTER); do
  echo "iterating over container $CONTAINER_ID"
  iterate $ITERATIONS $TIME $current_date $CONTAINER_ID
done
```

---
# Extender trial period do Datagrip

Requires version 2021.2.3 or less
``` Shell
#!/usr/bin/env bash

#rm -rf ~/.java/.userPrefs/prefs.xml
#rm -rf ~/.java/.userPrefs/jetbrains/prefs.xml


# Reset IntelliJ IDEA
rm -rf ~/.config/JetBrains/DataGrip2021.2/eval
# rm -rf ~/.config/JetBrains/IntelliJIdea*/options/other.xml
sed -i -E 's/<property name=\"evl.*\".*\/>//' ~/.config/JetBrains/DataGrip2021.2/options/other.xml
rm -rf ~/.java/.userPrefs/jetbrains/datagrip
```

---
# System-wide commands
``` Shell
#!/usr/bin/env bash

function preenvs() {
	source ~/workspace/<stub>/infrastructure/envs/pre-europe.env
	ssh-add -D
	$VAULT ssh -i ~/.ssh/id_rsa.pub
	ssh-add ~/.ssh/id_rsa
}

function pre_credentials() {
	source ~/workspace/<stub>/infrastructure/envs/pre-europe.env
	$VAULT rds --name pre-europe-rds-core
}

function prodenvs() {
	source ~/workspace/<stub>/infrastructure/envs/production-europe.env
	ssh-add -D
	$VAULT ssh -i ~/.ssh/id_rsa.pub
	ssh-add ~/.ssh/id_rsa
}

function prod_credentials() {
	source ~/workspace/<stub>/infrastructure/envs/production-europe.env
	$VAULT rds --name prod-europe-rds-core
}

function cienvs() {
	source ~/workspace/<stub>/infrastructure/envs/ci.env
	ssh-add -D
    	ssh-add ~/.ssh/id_rsa
}

function stub() {
	cd ~/workspace/<stub>/
}

function test_projects() {
	cd ~/workspace/test_projects/
}

function delete_dangling_local_branches() {
	git checkout master
	git branch -d $(git branch --merged|grep -v "^*" )
}

function pretty_hash() {
	git log -n1 --pretty=format:%h
}

alias ci1="ssh-add -D; ssh-add -t 3600 ~/.ssh/id_rsa; ssh pm@ci.stub.com"

alias ci2="ssh-add -D; ssh-add -t 3600 ~/.ssh/id_rsa; ssh pm@ci2.stub.com"
```
add to .bashrc
``` Shell
# source custom functions amd alias
source ~/Desktop/system-wide_commands.sh
```

---
# last git commit hash
``` Shell
git log -n 1 --pretty=format:%h
```

---
# remove dangling branches
``` Shell
git checkout master
git branch -d $(git branch --merged|grep -v "^*" )
```
or add to git config
```Shell
git config fetch.prune true
```

---
# change PS1 to add git branch to terminal
on .bashrc
``` Shell
PS1="\[\e[1;32m\]\u@\h\[\e[1;34m\] \w\[\e[3;33m\]\$(__git_ps1)\[\e[1;34m\]\n\$\[\e[0m\] "
```
