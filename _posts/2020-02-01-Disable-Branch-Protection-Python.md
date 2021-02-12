---
layout: post
title:  "Automating Gitlab with Python"
date:   2020-01-01
image: /images/posts/python.jpg
tags: [python,git,gitlab,api,lambda,devops,scripts]
---

I enjoy using Gitlab for internal lab environments as it has a robust api and can be automated using things like [`terraform`](https://registry.terraform.io/providers/gitlabhq/gitlab/latest/docs). One of things I normally use it for is creating represntaive environments with things like hooks enabled. This might looks like an example CI system thats fully built , and having a student have precreated content. However to create content in gitlab out of the box , or run something like `git push --mirror` you have to disable the branch protection. 

<!--more-->

<table>
    <caption>Versions tested</caption>
    <tbody>
        <tr>
            <th>Software</th>
            <th>Version</th>
            <th>OS</th>
        </tr>
        <tr>
            <td>Gitlab</td>
            <td>13.8</td>
            <td>AWS Lambda</td>
        </tr>
        <tr>
            <td>Python</td>
            <td>3</td>
            <td>AWS Lambda</td>
        </tr>
    </tbody>
</table>


  
`disable_branch_protection.py`
  

```python
#!/usr/bin/env python    
# Usage: GITLAB_PASSWORD=thepassword ./disable_branch_protection.py    

import gitlab    
import requests    
import json    
import os    
import requests    

GITLAB_GROUP   = 'homeops-tech'    
GITLAB_ADMIN   = 'root'    
GITLAB_API_URL = 'https://gitlab.homeops.tech:443'    

def get_gitlab_token(username,password):    
  session = requests.Session()    
  session.verify = '/etc/gitlab/ssl/gitlab.homeops.tech.crt'    
  session.keep_alive = True    
  # https://docs.gitlab.com/ce/api/oauth2.html#resource-owner-password-credential    
  url = 'https://gitlab.homeops.tech/oauth/token'    
  payload = {'username': username,    
             'password': password,    
             'grant_type': 'password'    
            }    
  r = session.post(url, data=payload, verify = '/etc/gitlab/ssl/gitlab.homeops.tech.crt')    
  oauth = json.loads(r.text)    
  api_token = oauth['access_token']    
  session.close    
  return api_token    

# Python lib doesn't support this after key creation
# https://docs.gitlab.com/ee/api/deploy_keys.html#enable-a-deploy-key
# This allows students to push to this repo with deploy key
def update_ssh_key():
  project = gl.projects.get('homeops-tech/control-repo')
  url = '%s/api/v4/projects/%s' % (GITLAB_API_URL,project.id)
  payload = {'default_branch': 'master'}
  headers = {'Authorization': 'Bearer %s' % api_token }
  r = requests.put(url, data=payload, verify=False, headers=headers)
  if r.status_code == 200:
    print "\tUpdated Deploy Key push access"
  else:
    print "\tUnable to update deploy key access"


api_token = get_gitlab_token(GITLAB_ADMIN,os.environ['GITLAB_PASSWORD'])    
gl = gitlab.Gitlab(GITLAB_API_URL, oauth_token=api_token,ssl_verify=False)    
project = gl.projects.get('homeops-tech/control-repo')    

branch = project.branches.get('master')    
branch.unprotect()
```

This script will use the local SSL certificate to validate the connection. You can pusht the password via 

```shell
GITLAB_PASSWORD=thepassword ./disable_branch_protection.py
```

The Python module supports many other options for automating common tasks

## Working with Branches 


```python
def create_student_branches(n):
  try:
    branch = project.branches.create({'branch': "student%s" % n,
                                    'ref': 'production'})
  except gitlab.exceptions.GitlabCreateError:
    print "\tstudent%s branch already exists" % n

def protect_branch():
  branch = project.branches.get('production')
  branch.protect()
```

This example code can create a branch and protect it.

## Working with Users

Add a user to the developer group.

```python
# Add the users to the puppet group as a developer
def add_students_to_group(n):
  try:
    student = gl.users.list(username="student%s" % n)[0]
    member = group.members.create({'user_id': student.id,
                                 'access_level': gitlab.DEVELOPER_ACCESS})
  except gitlab.exceptions.GitlabCreateError:
     print "\tstudent%s already a member of group" % n
```

Create a user

```python
# Create the student accounts
def create_students(n):
  try:
    user = gl.users.create({'email': "student%s@homeops.tech" % n,
                          'password': 'learn',
                          'username': "student%s" % n,
                          'skip_confirmation': True,
                          'name': "student %s" % n})
  except gitlab.exceptions.GitlabCreateError:
    print "\tstudent%s already created" % n
    # Skip the confirmation if user already created
    student = gl.users.list(username="student%s" % n)[0]
    student.skip_confirmation = True
    student.save()
```

Create a repo

```python
def create_module_repo(n):
  student = gl.users.list(username="student%s" % n)[0]
  try:
    user_project = student.projects.create({'name': 'time',
      'description': 'Example Puppet Module',
    })
  except gitlab.exceptions.GitlabCreateError:
    print "\t%s already has a time module created" % student.name
    user_project  = student.projects.list(name=MODULE_REPO)[0]
```

## Creating a new project

You can create projects and enable a deploy key for them

```python
api_token = get_gitlab_token(GITLAB_ADMIN,os.environ['GITLAB_PASSWORD'])
gl = gitlab.Gitlab(GITLAB_API_URL, oauth_token=api_token,ssl_verify=False)
project = gl.projects.get('homeops-tech/control-repo')
key = project.keys.create({'title': 'Student SSH Key',
                           'key': open('/root/.ssh/id_rsa.pub').read(),
                           'can_push': True,})
```

## Create a webhook for a project

```python
def add_project_hook(module_name):
  project = gl.projects.get('homeops-tech/%s' % module_name)
  project.hooks.list()
  project.hooks.create({'url': 'https://puppet.homeops.tech:8170/code-manager/v1/webhook?type=gitlab&token=%s' % (rbac_token),
    'push_events': 1, 'enable_ssl_verification': False})
```

This is an exmple of using Gitlab to configure a webhook for a given module repo. This example is using code manager with Puppet Enterprise
You can retreive a RBAC token with the following example code:


```python
def get_rbac_token():
  data = json.dumps({'login': 'admin',
                     'password': 'defaultpassword',
                     'label' : 'gitlab_webhook',
                     'description' : 'This is the token used for the Gitlab Webhooks',
                     'lifetime': '0'})
  r = requests.post('https://puppet.homeops.tech:4433/rbac-api/v1/auth/token',
    data=data,
    verify=False,
  )
  return json.loads(r.text)['token']
```

This example calls out to the RBAC API in puppet enterprise to get the Gitlab Token for code manager.

## Import a project

Gitlab has the ability to export a repo with all its metadata. You can then import this template via a tar.gz. file.

```shell
output = gl.projects.import_project(open('/tmp/export.tgz', 'rb'), 'control-repo',group.id)
# Get a ProjectImport object to track the import status
project_import = gl.projects.get(output['id'], lazy=True).imports.get()
while project_import.import_status != 'finished':
    time.sleep(1)
    project_import.refresh()
```

