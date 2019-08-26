  ## Start of Files

  You should start your scripts with some comments explaining what the script's purpose does (and an example usage, if necessary), followed by `---` with blank lines around it, then followed by the rest of the script.

  ```yaml
  #bad
  - name: 'Ensure nginx status'
    service:
      enabled: true
      name: 'nginx'
      state: '{{ state }}'
    become: true

  #good
  # Example usage: ansible-playbook -e state=started playbook.yml
  # This playbook changes the state of nginx service

  ---

  - name: 'Ensure nginx status'
    service:
      enabled: true
      name: 'nginx'
      state: '{{ state }}'
    become: true
  ```

  ### Why?

  This makes it easier to quickly find out the purpose/usage of a script, either by opening the file or using the `head` command.

  ## End of Files

  You should always end your files with a newline.

  ### Why?

  This is common Unix best practice, and avoids any prompt misalignment when printing files in a terminal.

## Quotes

**You should always quote strings** and **prefer single quotes over double quotes**.

```yaml
# bad
- name: Start nginx service
  service:
    name: nginx
    state: started
    enabled: true
  become: true

# good
- name: 'Start nginx service'
  service:
    name: 'nginx'
    state: 'started'
    enabled: true
  become: true
```

You should use double quotes when:
1. they are nested within single quotes (e.g. Jinja map reference)
2. when your string requires escaping characters (e.g. using "\n" to represent a newline)

```yaml
# double quotes w/ nested single quotes
- name: 'Start all services'
  service:
    name: '{{ item["name"] }}''
    state: 'started'
    enabled: true
  loop: '{{ services }}'
  become: true

# double quotes to escape characters
- name 'Print some text with special characters'
  debug:
    msg: "This text is on\ntwo lines"
```

Use the "folded scalar" style for really long strings and omit all special quoting. 
```yaml
# folded scalar style
- name: 'Print service info for all services'
  debug:
    msg: >
      Robot {{ item['name'] }} is {{ item['status'] }} and in {{ item['az'] }}
      availability zone with {{ item['number_of_processes'] }} processes running.
  with_items: '{{ services }}'
```

You should avoid quoting:
1. booleans (e.g. true/false)
2. numbers (e.g. 42)
3. things referencing the local Ansible environemnt e.g.
  - boolean logic
  - names of variables we are assigning values to

```yaml
# don't quote booleans/numbers
- name: 'Download example.com homepage'
  get_url:
    dest: '/tmp'
    timeout: 60
    url: 'https://example.com'
    validate_certs: true


# variables example
- name: 'Set a variable'
  set_fact:
    my_var: 'test'

# variables example with a conditional
- name: 'Print my_var'
  debug:
    var: my_var
  when: ansible_os_family == 'Darwin'

```

### Why?

Even though strings are the default type for YAML, syntax highlighting looks better when explicitly set types. This also helps troubleshoot malformed strings when they should be properly escaped to have the desired effect.

## Booleans

Use `true/false` to specify a boolean value.

```yaml
# bad
- name: 'Restart nginx service'
  service:
    name: 'nginx'
    state: 'restarted'
    enabled: 1
  become: 'yes'
 
# good
- name: 'Restart nginx service'
  service:
    name: 'nginx'
    state: 'restarted'
    enabled: true
  become: true
```

### Why?
There are many different ways to specify a boolean value in ansible, `True/False`, `true/false`, `yes/no`, `1/0`. We prefer to skick to one which is `true/false`.

## Key value pairs

Use only one space after the colon when designating a key value pair

```yaml
# bad
- name : 'Start nginx servce'
  service:
    name    : 'nginx'
    state   : 'restarted'
    enabled : true
  become : true


# good
- name: 'Start nginx servce'
  service:
    name: 'nginx'
    state: 'restarted'
    enabled: true
  become: true
```

**Always use the map syntax,** regardless of how many pairs exist in the map.

```yaml
# bad
- name: 'Create nginx config directory'
  file: 'path=/etc/nginx/conf.d state=directory mode=0755 owner=nginx group=nginx'
  become: true
  
- name: 'Deploy example website config'
  copy: 'dest=/etc/nginx/conf.d/example.conf template=example.conf.j2'
  become: true
  
# good
- name: 'Create nginx config directory
  file:
    path: '/etc/nginx/conf.d'
    owner: 'nginx
    group: 'nginx'
    mode: '0755'
    state: 'directory'
  become: true
  
- name: 'Deploy example website config'
  template:
    src: 'example.conf.j2'
    dest: '/etc/nginx/conf.d/example.conf'
  become: true
```

### Why?

It's easier to read and it's not hard to do. It reduces changeset collisions for version control.

## Sudo
Use the `become` syntax when designating that a task needs to be run with `sudo` privileges

```yaml
#bad
- name: 'Deploy example website config'
  template:
    src: 'example.conf.j2'
    dest: '/etc/nginx/conf.d/example.conf'
  sudo: true
 
# good
- name: 'Deploy example website config'
  template:
    src: 'example.conf.j2'
    dest: '/etc/nginx/conf.d/example.conf'
  become: true
```
### Why?
Using `sudo` was deprecated at [Ansible version 1.9.1](https://docs.ansible.com/ansible/latest/user_guide/become.html)

### Playbook structure

#### Play structure

Play should have the following order of declaration:
1. hosts
2. host options in alphabetical order
3. pre_tasks (optional)
4. roles
5. tasks (optional)
6. post_tasks (optional)


```yaml

- hosts: 'webservers'
  remote_user: 'centos'
  vars:
    timezone: 'Asia/Yerevan'
    
  pre_tasks:
    - name: 'Set the timezone to {{ timezone }}'
      lineinfile:
        dest: '/etc/environment'
        line: 'TZ={{ timezone }}'
        state: 'present'
      become: true
  roles:
    - role: 'nginx'
      vars:
        nginx_service_enabled: true
  tasks:
    - name: 'Ensure that nginx service is started'
      service:
        name: 'nginx'
        state: 'started'
      become: true
  post_tasks:
    - name: 'Notify everyone'
      mail:
        subject: 'System {{ ansible_hostname }} has been successfully provisioned.
      delegate_to: 'localhost'
```

#### Task structure

A task should have the following order of declaration:
1. name
2. module and its parameters
3. loop operators
4. conditional operators
5. other options (e.g. become, ignore_errors, register)
6. tags


```yaml
# (TODO: add an example with register in variables section)
- name: 'Ensure that all the packages are installed'
  apt:
    name: '{{ item }}'
    state: 'present'
  loop:
    - 'nginx'
    - 'vim'
  ignore_errors: true
  register: output
  when: ansible_os_family == 'Debian'
  tags:
    - 'package'
```


### Why?

A proper definition for how to order these maps produces consistent and easily readable code.

## Include Declaration

For `include`/`import` statements, make sure to quote filenames and only use blank lines between `include`/`import` statements if they are multi-line (e.g. they have tags).

```yaml
# bad
- include: other_file.yml

- include: 'second_file.yml'

- include: third_file.yml tags=third

# good

- include: 'other_file.yml'
- include: 'second_file.yml'

- include: 'third_file.yml'
  tags: 'third'
```

### Why?

This tends to be the most readable way to have `include`/`import` statements in your code.

## Spacing

You should have blank lines between plays, between two task blocks, and between host and include blocks. When indenting, you should use 2 spaces to represent sub-maps.

### Why?

This produces nice looking code that is easy to read.

## Task/Play names

Names should be written in sentence case.

```yaml
# bad
- name: 'This Is A Bad Name'
- name: 'this is also a bad name'
- name: 'ANOTHER BAD NAME'
  
# good
- name: 'This is a good name'
```

### Why?

This produces nice a looking output.

## Variable Names

Use `snake_case` for variable names in your scripts.

```yaml
# bad
- name: 'Set some facts'
  set_fact:
    myBoolean: true
    myint: 20
    MY_STRING: 'test'

# good
- name: 'Set some facts'
  set_fact:
    my_boolean: true
    my_int: 20
    my_string: 'test'
```

### Why?

Ansible uses `snake_case` for module names so it makes sense to extend this convention to variable names.
