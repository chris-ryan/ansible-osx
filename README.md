# Basic setup of fresh MacOS for dev using ansible & Homebrew

Started with a fresh install of MacOS Sierra 10.12.16
Tested to work with ansible: 2.3.1.0 & Homebrew 1.3.0

# Install dependencies

  - Download and install Command Line Tools for Xcode
    ```sh
    xcode-select --install 
    ```
  - Install Homebrew, ansible & git
    ```sh
    /usr/bin/ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
    brew install ansible
    brew install git
    ```
    
# Getting/creating the ansible playbook

You can download the ansible playbook from github, edit the files within `roles/common/vars``` to add your preferred packages, then skip to Running the playbook 
or, create it from scratch using the following instructions:

### Create the playbook folder and inventory
```sh
mkdir -p playbook
cd playbook
```
The inventory file tells ansible what machines to deploy to. Because this is a local, standalone deployment, ours is minimal:
```sh
vim inventory
```
```yaml
[local-mac]
Localhost
```
### Create the playbook master file
```sh
vim localhost.yml
```

- 

    ```yaml 
    - hosts: localhost
      connection: local
      gather_facts: no
    ```

> Note: We could just stick all of our tasks and variables in this file but for future extensibility we'll break them up into roles, with separate variables and tasks folders

### Define roles
- Make the roles folder
    ```sh 
    mkdir roles
    cd roles
    ```
	• Make the common role (for installs common to all users)
    ```sh
    mkdir common
    cd common
    mkdir tasks vars
    ```
> Note: We've only defined one role (common) here but down the track, if you have different user classes you'd combine multiple roles for different specialisations. (like nested manifests in munki)

### Define variables and tasks
- Make the brew variables file
```sh
vim vars/brew_packages.yml
```
-
    ```yaml 
    ---
	homebrew_taps:
	 - caskroom/cask
			
	homebrew_packages:
	 - { name: node, state: latest, install_options: without-npm }
	 - { name: wget, state: present }
	 - { name: dockutil, state: latest }
			
	homebrew_cask_packages:
      - { name: google-chrome }
      - { name: slack }
      - { name: the-unarchiver }
      - { name: visual-studio-code }
      - { name: vlc }
    ```
> Note: you can find the names for other available casks at https://caskroom.github.io/search

- Make the dockutils variables file
`vim vars/dockutils_items.yml`
-
    ```yaml 
    ---
	dockitems_to_remove:
	 - Mail
	 - Reminders
	 - Photos
	 - FaceTime
	 - iTunes
	 - iBooks
	 - App Store
			
	dockitems_to_persist:
	 - name: Google Chrome
	   path: "/Applications/Google Chrome.app"
       pos: 1
	 - name: Slack
	   path: "/Applications/Slack.app"
       pos: 2
	 - name: Terminal
	   path: "/Applications/Utilities/Terminal.app"
       pos: 3
    ```
- Make the brew tasks file (for installing homebrew & homebrew cask apps)
`vim tasks/brew.yml`
-
    ```yaml 
	- name: homebrew add repo
	  homebrew_tap: tap={{ item }} state=present
	  with_items: "{{ homebrew_taps }}"
			
	- name: Update homebrew
	  homebrew: update_homebrew=yes
			
	# brew
	- name: Install homebrew packages.
	  homebrew: >
	    name: ["{{ item.name | default(item) }}"]
	    install_options: ["{{ item.install_options | default(omit) }}"]
	    state: "{{ item.state | default(homebrew_default_state) }}"
	  with_items: "{{ homebrew_packages }}"
	- name: Create directory for packages info.
	  file: path=brew_info state=directory
	- name: Save brew info.
	  shell: brew info "{{ item.name }}" > brew_info/{{ item.name }}
	  with_items: "{{ homebrew_packages }}"
			
	# cask
	- name: Install cask packages.
	  homebrew_cask:
	    name: "{{ item.name }}"
	    state: "{{ item.state|default('present') }}"
	  with_items: "{{ homebrew_cask_packages }}"
	- name: cask mkdir for package info
	  file: path=cask_info state=directory
	- name: cask save info
	  shell: brew cask info {{ item.name }} > cask_info/{{ item.name }}
	  with_items: "{{ homebrew_cask_packages }}"
    ```
  - Make the dockutils tasks file (for cleaning up the dock)
  - `vim tasks/dockutils.html`
  - 			
        ---
	    - name: Remove unwanted items from Dock
		  shell: dockutil --remove '{{ item }}'
		  ignore_errors: true
		  with_items: "{{ dockitems_to_remove  }}"
			
		- name: Check if items in dock exist
		  shell: dockutil --find '{{ item.name }}' || dockutil --add '{{ item.path }}'
		  with_items: "{{ dockitems_to_persist }}"
			
		- name: Fix order
		  shell: dockutil --move '{{ item.name }}' --position {{ item.pos }}
		  with_items: "{{ dockitems_to_persist }}"
  
  > Note: Since ansible v2.2, ```with_items``` requires explicit jinja2 wrapping

	• Make the master tasks file
	
	  - ```vim tasks/main.yml```
  - 			
        ---
	    - name: Remove unwanted items from Dock
		  shell: dockutil --remove '{{ item }}'
		  ignore_errors: true
		
		- 
		
	```yaml
	- name: Load all variable files
	 include_vars:
	   dir: '../vars'
		
	- include: brew.yml
	- include: dockutils.yml
	```

## Running the playbook
- From the root of the playbook folder
- 
    ```sh
    ansible-playbook -i inventory --ask-become-pass localhost.yml
    brew cask cleanup
    ```

## References


* [Medium] - Automation of installation on Mac with ansible
* [Ansible-docs-homebrew] - Syntax and use of the ansible homebrew module
* [Ansible-docs-best-practices] - Markdown parser done right. Fast and easy to extend.
* [Ansible-docs-playbooks] - great UI boilerplate for modern web apps
* [Ansible-docs-loops] - evented I/O for the backend


   [Medium]: <https://medium.com/@kojiitp/automation-of-installation-on-mac-w-ansible-21354cce0d7b>
   [Ansible-docs-homebrew]: <http://docs.ansible.com/ansible/latest/homebrew_module.html>
   [Ansible-docs-best-practices]: <http://docs.ansible.com/ansible/latest/playbooks_best_practices.html>
   [Ansible-docs-playbooks]: <http://docs.ansible.com/ansible/latest/playbooks_intro.html>
   [Ansible-docs-loops]: <http://docs.ansible.com/ansible/latest/playbooks_loops.html>

   [PlDb]: <https://github.com/joemccann/dillinger/tree/master/plugins/dropbox/README.md>
   [PlGh]: <https://github.com/joemccann/dillinger/tree/master/plugins/github/README.md>
   [PlGd]: <https://github.com/joemccann/dillinger/tree/master/plugins/googledrive/README.md>
   [PlOd]: <https://github.com/joemccann/dillinger/tree/master/plugins/onedrive/README.md>
   [PlMe]: <https://github.com/joemccann/dillinger/tree/master/plugins/medium/README.md>
   [PlGa]: <https://github.com/RahulHP/dillinger/blob/master/plugins/googleanalytics/README.md>
