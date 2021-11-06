# Script to provision LDAP users and groups in AWS SSO using SCIM

* Author: Gustavo Lichti Mendonça
* Mail: gustavo.lichti@gmail.com
* This Code: https://github.com/lichti/aws-sso-provisioning

## Why i need todo this

Because I bought a commercial tool from Cyberark to do this and when the tool stopped working, it doesn't support customers.

## Dependencies installing


```bash
%%bash
pip install requests
pip install pyyaml
pip install python-ldap
```

## Imports


```python
import requests
import json
import ldap
import configparser
```

## Load config file with credentials

Read more about configparser: https://docs.python.org/3/library/configparser.html

Config Teamplate:

```text
[AWS-SSO-SCIM]
base_url = https://scim.us-east-1.amazonaws.com/YOUR-AWS-SSO-ID/scim/v2/
bearertoken = YOUR-AWS-SSO-BEARERTOKEN


[LDAP]
LDAP_SERVER = ldap://YOUR-LDAP-ADDRESS
BASE_DN_AWS_SSO_GROUPS = OU=AWS-SSO,OU=PROVISIONING,OU=GROUPS,DC=foo,DC=bar,DC=local
BASE_DN_USERS = OU=USERS,OU=GROUPS,DC=foo,DC=bar,DC=local
LDAP_LOGIN = my-user@foo.bar.local
LDAP_PASSWORD = MY-SUPER-STRONG-PASSWORD
```


```python
config = configparser.ConfigParser()
config.read('aws-sso-scim-ldap.ini')
```

## LDAP CONFIG


```python
LDAP_SERVER = config['LDAP']['LDAP_SERVER']
LDAP_LOGIN = config['LDAP']['LDAP_LOGIN']
LDAP_PASSWORD = config['LDAP']['LDAP_PASSWORD']
```

## SCIM AWS SSO CONFIG

Learn more about AWS SSO SCIM:
* https://docs.aws.amazon.com/singlesignon/latest/developerguide/supported-apis.html


```python
base_url = config['AWS-SSO-SCIM']['base_url']
bearertoken = config['AWS-SSO-SCIM']['bearertoken']
users_url = f"{base_url}Users"
headers_auth = {"Authorization": f"Bearer {bearertoken}", "Content-type": "application/json"}
```

## HTTP helpers

Basic http methods helpers (get, post, put, patch, delete)

Recommended reading: 
* https://datatracker.ietf.org/doc/html/rfc7231#section-4.3
* https://datatracker.ietf.org/doc/html/rfc7644#section-3.2

### Get


```python
def get(path=None, params=None):
    return requests.get(f"{base_url}{path}",headers=headers_auth, params=params)
```

### Post


```python
def post(path=None, params=None, data=None):
    return requests.post(f"{base_url}{path}",headers=headers_auth, data=data)
```

### Put


```python
def put(path=None, params=None, data=None):
    return requests.put(f"{base_url}{path}",headers=headers_auth, data=data)
```

### Patch


```python
def patch(path=None, params=None, data=None):
    return requests.patch(f"{base_url}{path}",headers=headers_auth, data=data)
```

### Delete


```python
def delete(path=None):
    return requests.delete(f"{base_url}{path}",headers=headers_auth)
```

## SCIM helpers

Basic SCIM methods helpers

Learn more abut SCIM:
* https://datatracker.ietf.org/doc/html/rfc7642
* https://datatracker.ietf.org/doc/html/rfc7643
* https://datatracker.ietf.org/doc/html/rfc7644
* https://openid.net/specs/fastfed-scim-1_0-02.html#rfc.section.4

### Users

#### CreateUser

https://datatracker.ietf.org/doc/html/rfc7644#section-3.3


```python
def createUser(userName=None,familyName=None,givenName=None,displayName=None,email=None,
               preferredLanguage="en-US",locale="en-US",timezone="America/Sao_Paulo",active=True):
    if userName and familyName and givenName and displayName and email:
        data = {
            "userName": f"{userName}",
            "name": {
                "familyName": f"{familyName}",
                "givenName": f"{givenName}",
            },
            "displayName": f"{displayName}",
            "emails": [
                {
                    "value": f"{email}",
                    "type": "work",
                    "primary": True
                }
            ],
            "preferredLanguage": f"{preferredLanguage}",
            "locale": f"{locale}",
            "timezone": f"{timezone}",
            "active": f"{active}",
        }
        res = post(path=f"Users", data=json.dumps(data))
        if res.status_code == 201:
            return json.loads(res.text)['id']
        else:
            print(res.content)
```

#### ListUsers

https://datatracker.ietf.org/doc/html/rfc7644#section-3.4.1


```python
def listUsers(params=None):
    res = get(path='Users',params=params)
    if res.status_code == 200:
        users = json.loads(res.text)
        return users
```

#### HasUserByUsername

https://datatracker.ietf.org/doc/html/rfc7644#section-3.4.1


```python
def hasUserByUsername(userName=None):
    if userName:
        users = listUsers(f'filter=userName eq "{userName}"')['Resources']
        for u in users:
            if u['userName'] == userName:
                return True
    return False
```

#### GetUserIDByUsername

https://datatracker.ietf.org/doc/html/rfc7644#section-3.4.1


```python
def getUserIDByUsername(userName=None):
    if userName:
        users = listUsers(f'filter=userName eq "{userName}"')['Resources']
        for u in users:
            if u['userName'] == userName:
                return u['id']
```

#### GetUser

https://datatracker.ietf.org/doc/html/rfc7644#section-3.4.1


```python
def getUser(user_id=None):
    if user_id:
        res = get(path=f"Users/{user_id}")
        if res.status_code == 200:
            return json.loads(res.text)
```

#### ReplaceUser

https://datatracker.ietf.org/doc/html/rfc7644#section-3.5.1


```python
def replaceUser(user_id=None,userName=None,familyName=None,givenName=None,displayName=None,email=None,
               preferredLanguage="en-US",locale="en-US",timezone="America/Sao_Paulo",active=True):
    if user_id and userName and familyName and givenName and displayName and email:
        data = {
            "id": f"{user_id}",
            "userName": f"{userName}",
            "name": {
                "familyName": f"{familyName}",
                "givenName": f"{givenName}",
            },
            "displayName": f"{displayName}",
            "emails": [
                {
                    "value": f"{email}",
                    "type": "work",
                    "primary": True
                }
            ],
            "preferredLanguage": f"{preferredLanguage}",
            "locale": f"{locale}",
            "timezone": f"{timezone}",
            "active": f"{active}",
        }
        res = put(path=f"Users/{user_id}", data=json.dumps(data))
        if res.status_code == 200:
            return json.loads(res.text)['id']
        else:
            print(res.content)
```

#### UpdateUser - I need improve this...

https://datatracker.ietf.org/doc/html/rfc7644#section-3.5.2


```python
def updateUser(user_id=None, data=None):
    return json.loads(patch(path=f"Users/{user_id}", data=data).text)
```

#### DeleteUser

https://datatracker.ietf.org/doc/html/rfc7644#section-3.6


```python
def deleteUser(user_id=None):
    res = delete(path=f"Users/{user_id}")
    if res.status_code == 204:
        return True
    return False
```

### Groups

#### CreateGroup

https://datatracker.ietf.org/doc/html/rfc7644#section-3.3


```python
def createGroup(groupName=None):
    if groupName:
        data = {"displayName": f"{groupName}"}
        res = post(path=f"Groups", data=json.dumps(data))
        if res.status_code == 201:
            return json.loads(res.text)['id']
        else:
            print(res.content)
```

#### ListGroups

https://datatracker.ietf.org/doc/html/rfc7644#section-3.4.1


```python
def listGroups(params=None):
    res = get(path='Groups',params=params)
    if res.status_code == 200:
        groups = json.loads(res.text)
        return groups
```

#### HasGroupByName

https://datatracker.ietf.org/doc/html/rfc7644#section-3.4.1


```python
def hasGroupByName(groupName=None):
    if groupName:
        groups = listGroups(f'filter=displayName eq "{groupName}"')['Resources']
        for g in groups:
            if g['displayName'] == groupName:
                return True
    return False
```

#### GetGroupIDByName

https://datatracker.ietf.org/doc/html/rfc7644#section-3.4.1


```python
def getGroupIBByName(groupName=None):
    if groupName:
        groups = listGroups(f'filter=displayName eq "{groupName}"')['Resources']
        for g in groups:
            if g['displayName'] == groupName:
                return g['id']
```

#### GetGroup

https://datatracker.ietf.org/doc/html/rfc7644#section-3.4.1


```python
def getGroup(group_id=None):
    if group_id:
        res = get(path=f"Groups/{group_id}")
        if res.status_code == 200:
            return json.loads(res.text)
```

#### UpdateGroup

https://datatracker.ietf.org/doc/html/rfc7644#section-3.5.2


```python
def updateGroup(group_id=None, operation=None, members=None):
    if group_id and operation and members:
        data = {
            "schemas":["urn:ietf:params:scim:api:messages:2.0:PatchOp"],
            "Operations":[
                {
                    "op": f"{operation}",
                    "path": "members",
                    "value":[{"value": f"{member}"} for member in members]
                }
            ]
        }
        res = patch(path=f"Groups/{group_id}", data=json.dumps(data))
        if res.status_code == 204:
            return True
        else:
            print(res.content)
            return False
```

#### DeleteGroup

https://datatracker.ietf.org/doc/html/rfc7644#section-3.6


```python
def deleteGroup(group_id=None):
    res = delete(path=f"Groups/{group_id}")
    if res.status_code == 204:
        return True
    return False
```

## AWS SSO SCIM SYNC WITH LDAP

### Connect to LDAP


```python
connect = ldap.initialize(LDAP_SERVER)
connect.set_option(ldap.OPT_REFERRALS, 0)  # to search the object and all its descendants
connect.simple_bind_s(LDAP_LOGIN, LDAP_PASSWORD)
```

### Retrieve from LDAP all AWS SSO Provisioning Groups


```python
BASE_DN_AWS_SSO_GROUPS = config['LDAP']['BASE_DN_AWS_SSO_GROUPS']
groups=connect.search_s(BASE_DN_AWS_SSO_GROUPS, ldap.SCOPE_SUBTREE, 'ObjectClass=Group', ['cn','dn'])
```

### get_group_users

Method to get all nested users from groups

userAccountControl codes:
* 514 = 512 + 2
* 546 = 512 + 32 + 2
* 66050 = 65536 + 512 + 2
* 66082 = 65536 + 512 + 32 + 2

More:
* 2:     ACCOUNTDISABLE
* 32:    PASSWD_NOTREQD
* 512:   NORMAL_ACCOUNT
* 65536: DONT_EXPIRE_PASSWORD

Read more: https://docs.microsoft.com/pt-br/troubleshoot/windows-server/identity/useraccountcontrol-manipulate-account-properties

If necesssary read more about bytestring to better undertand this section:
* https://stackoverflow.com/questions/6224052/what-is-the-difference-between-a-string-and-a-byte-string
* https://stackoverflow.com/questions/22824539/what-is-a-python-bytestring
* https://stackoverflow.com/questions/606191/convert-bytes-to-a-string

In this method, we use the LDAP OID operator to perform a nested search or a recursive search, whichever you prefer. Study more about this here: https://docs.microsoft.com/pt-br/windows/win32/adsi/search-filter-syntax#operators


```python
def get_group_users(conn,base_dn,dn):
    membersOf=conn.search_s(base_dn,
                            ldap.SCOPE_SUBTREE,
                            f'(&(objectClass=user)(memberof:1.2.840.113556.1.4.1941:={dn}))',
                            ['objectClass', 'userAccountControl', 'userPrincipalName', 'cn', 'mail', 'displayName', 'givenName', 'sn']
                           )
    members = []
    for member in membersOf:
        if not member[1]['userAccountControl'][0].decode('utf-8') in ['546', '66050', '66082']:
            if member[1]['userAccountControl'][0].decode('utf-8') in ['514']:
                member_status = False
            else:
                member_status = True
            members.append({'member_dn': member[0],
                            'member_cn': member[1]['cn'][0].decode('utf-8'),
                            'member_mail': member[1]['mail'][0].decode('utf-8'),
                            'member_upn': member[1]['userPrincipalName'][0].decode('utf-8'),
                            'member_displayName': member[1]['displayName'][0].decode('utf-8'),
                            'member_givenName': member[1]['givenName'][0].decode('utf-8'),
                            'member_sn': member[1]['sn'][0].decode('utf-8'),
                            'member_status': member_status,
                            'member_userAccountControl': member[1]['userAccountControl']
                           })
    return members

```

### groups_with_members  

List of all groups and their members (rich data)


```python
BASE_DN_USERS = config['LDAP']['BASE_DN_USERS']
groups_with_members = []
for group in groups:
    group_members = get_group_users(connect,BASE_DN_USERS,group[0])
    groups_with_members.append({'group_dn': group[0],
                        'group_cn': group[1]['cn'][0].decode('utf-8'),
                        'group_members': group_members})

```

### CreateOrUpdateUser

Method for creating or updating a user by SCIM provisioning


```python
def CreateOrUpdateUser(member=None):
    print(f"{member['member_displayName']} => {member['member_userAccountControl']} => {member['member_status']}")
    if not hasUserByUsername(member['member_upn']):
        print(f"--> Creating user {member['member_upn']} -> {member['member_displayName']}")
        ID = createUser(userName=member['member_upn'],
                        familyName=member['member_sn'],
                        givenName=member['member_givenName'],
                        displayName=member['member_displayName'],
                        email=member['member_mail'],
                        preferredLanguage="en-US",
                        locale="en-US",
                        timezone="America/Sao_Paulo",
                        active=member['member_status'])
        if ID:
            print(f"----> User created: {ID}")
        else:
            print("----> User create failed")
    else:
        ID = getUserIDByUsername(member['member_upn'])
        print(f"--> Updating user {member['member_upn']} -> {member['member_displayName']} -> {ID}")  
        if replaceUser(user_id=ID,
                       userName=member['member_upn'],
                       familyName=member['member_sn'],
                       givenName=member['member_givenName'],
                       displayName=member['member_displayName'],
                       email=member['member_mail'],
                       preferredLanguage="en-US",
                       locale="en-US",
                       timezone="America/Sao_Paulo",
                       active=member['member_status']):
            print("----> User updated")
        else:
            print("----> User update failed")
    return ID
```

### listOfUsernamesToIDS

Helper to create a list of IDs from a list of usernames. Need a dictionary to do the black magic to work


```python
def listOfUsernamesToIDS(usernames=None, usernames_dict=None):
    IDs=[]
    for username in usernames:
        if username in usernames_dict:
            IDs.append(usernames_dict[username])
    return IDs
```

### members_dict

Dict to store username => id

```{'username1': 'id1', 'username2': 'id2', 'username..n': 'id..n'}```


```python
members_dict={}
```

### members_unique

List of members dict

``` [{member1}, {member2}, {member..n}] ```


```python
members_unique=[]
```

### Populate members_dict and members_unique


```python
total_processed=0
for group in groups_with_members:
    if group['group_members']:
        for member in group['group_members']:
            total_processed=total_processed+1
            if not member in members_unique:
                members_unique.append(member)
                memberID = getUserIDByUsername(member['member_upn'])
                members_dict[member['member_upn']]=memberID
print(f"Groups: {len(groups_with_members)} | Processed members: {total_processed} | Unique members: {len(members_unique)}")


```

### Create or Update unique members


```python
cont=0
for member in members_unique:
    cont = cont +1
    print(f">{cont}/{len(members_unique)}")
    CreateOrUpdateUser(member)
```


```python
step=0
for group in groups_with_members:
    step=step+1
    group_name = group['group_cn'].upper()
    print(f"({step}/{len(groups_with_members)}) Working in the group: {group_name}")
    members=[]
    if group['group_members']:
        for member in group['group_members']:
            members.append(member['member_upn'])
    if members:
        if hasGroupByName(group_name):
            print(f"--> Group exists")
            IDs = listOfUsernamesToIDS(members,members_dict)
            GroupID = getGroupIBByName(group_name)
            if updateGroup(GroupID,"replace",IDs):
                print("----> Group members updated")
            else:
                print("----> Group members update failed")
        else:
            print(f"--> Creating group")
            Group_ID = createGroup(group_name)
            if Group_ID:
                print("----> Group created")
                IDs = listOfUsernamesToIDS(members,members_dict)
                if updateGroup(GroupID,"replace",IDs):
                    print("------> Group members updated")
                else:
                    print("------> Group members update failed")
            else:
                print("----> Group create failed")

```
