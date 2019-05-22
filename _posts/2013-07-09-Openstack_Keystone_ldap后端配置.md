---
layout: post
category : OpenStack
tagline: "regexisart"
tags : [OpenStack, keystone, LDAP]
---
{% include JB/setup %}

**如需转载，请标明原文出处以及作者**

*陈锐 RuiChen @kiwik*

*2013/07/09 23:16:50*

----------

*写在最前面：*

*以下内容在openstack G版2013.1代码和ubuntu 12.04 LTS环境验证*

*LDAP版本为openldap-2.4.28*

## 安装LDAP

```bash
apt-get install ldap-utils
apt-get install slapd
```

## 验证LDAP登录

ldap安装默认根据当前主机的域名，生成登陆的DN

查看 `/etc/hosts`

![][1]

所以这个环境上的登陆DN为 `cn=admin,dc=openstack,dc=org`

使用此命令验证是否配置成功

`ldapsearch -x -LLL -H ldap:/// -b dc=openstack,dc=org dn`

![][2]

## 修改LDAP的默认schema

LDAP的默认schema不能直接和openstack配合使用，有些openstack的用户、角色、租户需要的属性默认schema中没有，例如：enable，description等等，需要修改；其次，需要添加存储openstack相关模型（user，tenant，group，role，domain）的dn，以便保存数据。
我自己写了两个ldif文件，以便完成上面两件事，内容如下：

> modify.ldif

```txt

dn: cn={0}core,cn=schema,cn=config  
changetype: modify  
add: olcAttributeTypes  
olcAttributeTypes: {52}( 2.5.4.66 NAME 'enabled' DESC 'RFC2256: enabled of a group' EQUALITY booleanMatch SYNTAX 1.3.6.1.4.1.1466.115.121.1.7 SINGLE-VALUE )  
  
dn: cn={0}core,cn=schema,cn=config  
changetype: modify  
delete: olcObjectClasses  
olcObjectClasses: {7}( 2.5.6.9 NAME 'groupOfNames' DESC 'RFC2256: a group of names (DNs)' SUP top STRUCTURAL MUST ( member $ cn ) MAY ( businessCategory $ seeAlso $ owner $ ou $ o $ description ) )  
-  
add: olcObjectClasses  
olcObjectClasses: {7}( 2.5.6.9 NAME 'groupOfNames' DESC 'RFC2256: a group of names (DNs)' SUP top STRUCTURAL MUST ( member $ cn ) MAY ( businessCategory $ seeAlso $ owner $ ou $ o $ description $ enabled) )  
  
dn: cn={3}inetorgperson,cn=schema,cn=config  
changetype: modify  
delete: olcObjectClasses  
olcObjectClasses: {0}( 2.16.840.1.113730.3.2.2 NAME 'inetOrgPerson' DESC 'RFC2798: Internet Organizational Person' SUP organizationalPerson STRUCTURAL MAY ( audio $ businessCategory $ carLicense $ departmentNumber $ displayName $ employeeNumber $ employeeType $ givenName $ homePhone $ homePostalAddress $ initials $ jpegPhoto $ labeledURI $ mail $ manager $ mobile $ o $ pager $ photo $ roomNumber $ secretary $ uid $ userCertificate $ x500uniqueIdentifier $ preferredLanguage $ userSMIMECertificate $ userPKCS12 ) )  
-  
add: olcObjectClasses  
olcObjectClasses: {0}( 2.16.840.1.113730.3.2.2 NAME 'inetOrgPerson' DESC 'RFC2798: Internet Organizational Person' SUP organizationalPerson STRUCTURAL MAY ( audio $ businessCategory $ carLicense $ departmentNumber $ displayName $ employeeNumber $ employeeType $ givenName $ homePhone $ homePostalAddress $ initials $ jpegPhoto $ labeledURI $ mail $ manager $ mobile $ o $ pager $ photo $ roomNumber $ secretary $ uid $ userCertificate $ x500uniqueIdentifier $ preferredLanguage $ userSMIMECertificate $ userPKCS12 $ description $ enabled $ email ) )

```

将以上内容保存到对应的文件之后，执行如下命令：

`ldapmodify -c -Y EXTERNAL -H ldapi:/// -f modify.ldif`

> add.ldif

```txt
dn: ou=users,dc=openstack,dc=org  
objectClass: top  
objectClass: organizationalUnit  
  
dn: ou=projects,dc=openstack,dc=org  
objectClass: top  
objectClass: organizationalUnit  
  
dn: ou=roles,dc=openstack,dc=org  
objectClass: top  
objectClass: organizationalUnit  
  
dn: ou=groups,dc=openstack,dc=org  
objectClass: top  
objectClass: organizationalUnit  
  
dn: ou=domains,dc=openstack,dc=org  
objectClass: top  
objectClass: organizationalUnit  
```

将以上内容保存到对应的文件之后，执行如下命令：

`ldapadd -x -c -D"cn=admin,dc=openstack,dc=org" -w "Galax8800" -f add.ldif`

注意Galax8800是安装LDAP的时候，输入的root密码

## 修改keystone的配置文件

> /etc/keystone/keystone.conf

将Identity的后端配置为ldap

```ini
[identity]  
driver = keystone.identity.backends.ldap.Identity  
```

增加ldap段的配置，只需如下配置，其他可以使用默认值

```ini
[ldap]  
url = ldap://localhost  
user = cn=admin,dc=openstack,dc=org  
password = Galax8800  
suffix = dc=openstack,dc=org  
use_dumb_member = True  
allow_subtree_delete = False  
  
user_tree_dn = ou=users,dc=openstack,dc=org  
tenant_tree_dn = ou=projects,dc=openstack,dc=org  
role_tree_dn = ou=roles,dc=openstack,dc=org  
group_tree_dn = ou=groups,dc=openstack,dc=org  
domain_tree_dn = ou=domains,dc=openstack,dc=org  
```

注意：password = `Galax8800` 此密码为当时安装LDAP时，输入的root密码

配置完成之后，重启keystone

## 初始化keystone的基本用户

这个步骤网上的一些外国大牛已经提供了一些脚本，这里为了流程完整，就提供一个参考版本

> keystone_basic.sh

```bash
#!/bin/sh  

ADMIN_PASSWORD=Galax8800  
SERVICE_PASSWORD=Galax8800  
SERVICE_TENANT_NAME=${SERVICE_TENANT_NAME:-service}  
  
get_id () {  
    echo `$@ | awk '/ id / { print $4 }'`  
}  
  
# Tenants  
ADMIN_TENANT=$(get_id keystone tenant-create --name=admin)  
SERVICE_TENANT=$(get_id keystone tenant-create --name=$SERVICE_TENANT_NAME)  
  
  
# Users  
ADMIN_USER=$(get_id keystone user-create --name=admin --pass="$ADMIN_PASSWORD" --email=admin@domain.com)  
  
  
# Roles  
ADMIN_ROLE=$(get_id keystone role-create --name=admin)  
KEYSTONEADMIN_ROLE=$(get_id keystone role-create --name=KeystoneAdmin)  
KEYSTONESERVICE_ROLE=$(get_id keystone role-create --name=KeystoneServiceAdmin)  
  
# Add Roles to Users in Tenants  
keystone user-role-add --user-id $ADMIN_USER --role-id $ADMIN_ROLE --tenant-id $ADMIN_TENANT  
keystone user-role-add --user-id $ADMIN_USER --role-id $KEYSTONEADMIN_ROLE --tenant-id $ADMIN_TENANT  
keystone user-role-add --user-id $ADMIN_USER --role-id $KEYSTONESERVICE_ROLE --tenant-id $ADMIN_TENANT  
  
# The Member role is used by Horizon and Swift  
MEMBER_ROLE=$(get_id keystone role-create --name=Member)  
  
# Configure service users/roles  
NOVA_USER=$(get_id keystone user-create --name=nova --pass="$SERVICE_PASSWORD" --email=nova@domain.com)  
keystone user-role-add --tenant-id $SERVICE_TENANT --user-id $NOVA_USER --role-id $ADMIN_ROLE  
  
GLANCE_USER=$(get_id keystone user-create --name=glance --pass="$SERVICE_PASSWORD" --email=glance@domain.com)  
keystone user-role-add --tenant-id $SERVICE_TENANT --user-id $GLANCE_USER --role-id $ADMIN_ROLE  
  
QUANTUM_USER=$(get_id keystone user-create --name=quantum --pass="$SERVICE_PASSWORD" --email=quantum@domain.com)  
keystone user-role-add --tenant-id $SERVICE_TENANT --user-id $QUANTUM_USER --role-id $ADMIN_ROLE  
  
CINDER_USER=$(get_id keystone user-create --name=cinder --pass="$SERVICE_PASSWORD" --email=cinder@domain.com)  
keystone user-role-add --tenant-id $SERVICE_TENANT --user-id $CINDER_USER --role-id $ADMIN_ROLE  
```

将以上内容保存到对应的文件之后，执行如下命令：

`sh keystone_basic.sh`

## 验证keystone

如果上述流程都已经执行成功了，需要验证一下keystone中的数据，执行

`keystone user-list`

![][3]

`keystone role-list`

![][4]

`keystone tenant-list`

![][5]

`keystone user-role-list`

![][6]

注意：keystone user-role-list如果执行失败，没有关系，这不是LDAP配置的问题，是一个keystone-client的[bug](https://bugs.launchpad.net/python-keystoneclient/+bug/1058750)，通过Rest Client重新验证一下就可以了

## 查看LDAP中的数据结构

我们再来看一下LDAP中的数据结构，如果和下图一致，就没有问题了

![][7]

## 重启openstack的所有进程

配置keystone LDAP成功之后，需要重启openstack的所有进程，以nova为例，nova会缓存api-paste.ini中配置的admin_user的token，如果配置LDAP之前是使用的sql Identity后端，那么nova会使用这个缓存的admin_token去验证消息携带的token，keystone在判断admin_token是否可用时，会报错，造成认证失败，这个问题当时困扰了我很久，所以一定要重启openstack的所有进程，然后需要验证一下nova list，cinder list，glane image-list，quantum net-list是否可用。

## 将以上步骤浓缩为一个脚本

写了一个脚本将以上步骤全部实现

```bash
#!/bin/bash  
  
# ubunut 12.04 LTS keystone G 2013.1 openldap 2.4.28  
  
LDAP_PASS='Galax8800'  
  
KEYSTONE_CONF='/etc/keystone/keystone.conf'  
  
#step0 install ldap and set password  
function install_ldap()  
{  
    cat <<LDAP_PRESEED | debconf-set-selections   
slapd slapd/password1 password ${LDAP_PASS}  
slapd slapd/password2 password ${LDAP_PASS}  
LDAP_PRESEED  
  
    apt-get -y --force-yes install slapd  
    apt-get -y --force-yes install ldap-utils  
  
}  
  
#step1 check ldap login  
function check_ldap_login()  
{  
    local first_line=$(ldapsearch -x -LLL -H ldap:/// -b dc=openstack,dc=org dn | sed -n 1p)  
      
    if [ "${first_line}" == "dn: dc=openstack,dc=org" ]  
    then  
        echo "login success"  
    else  
        echo "login failed"  
        exit  
    fi  
}  
  
#step2 modify ldap schema  
function modify_ldap_schema()  
{  
    cat > ./modify.ldif <<EOF  
dn: cn={0}core,cn=schema,cn=config  
changetype: modify  
add: olcAttributeTypes  
olcAttributeTypes: {52}( 2.5.4.66 NAME 'enabled' DESC 'RFC2256: enabled of a group' EQUALITY booleanMatch SYNTAX 1.3.6.1.4.1.1466.115.121.1.7 SINGLE-VALUE )  
  
dn: cn={0}core,cn=schema,cn=config  
changetype: modify  
delete: olcObjectClasses  
olcObjectClasses: {7}( 2.5.6.9 NAME 'groupOfNames' DESC 'RFC2256: a group of names (DNs)' SUP top STRUCTURAL MUST ( member $ cn ) MAY ( businessCategory $ seeAlso $ owner $ ou $ o $ description ) )  
-  
add: olcObjectClasses  
olcObjectClasses: {7}( 2.5.6.9 NAME 'groupOfNames' DESC 'RFC2256: a group of names (DNs)' SUP top STRUCTURAL MUST ( member $ cn ) MAY ( businessCategory $ seeAlso $ owner $ ou $ o $ description $ enabled) )  
  
dn: cn={3}inetorgperson,cn=schema,cn=config  
changetype: modify  
delete: olcObjectClasses  
olcObjectClasses: {0}( 2.16.840.1.113730.3.2.2 NAME 'inetOrgPerson' DESC 'RFC2798: Internet Organizational Person' SUP organizationalPerson STRUCTURAL MAY ( audio $ businessCategory $ carLicense $ departmentNumber $ displayName $ employeeNumber $ employeeType $ givenName $ homePhone $ homePostalAddress $ initials $ jpegPhoto $ labeledURI $ mail $ manager $ mobile $ o $ pager $ photo $ roomNumber $ secretary $ uid $ userCertificate $ x500uniqueIdentifier $ preferredLanguage $ userSMIMECertificate $ userPKCS12 ) )  
-  
add: olcObjectClasses  
olcObjectClasses: {0}( 2.16.840.1.113730.3.2.2 NAME 'inetOrgPerson' DESC 'RFC2798: Internet Organizational Person' SUP organizationalPerson STRUCTURAL MAY ( audio $ businessCategory $ carLicense $ departmentNumber $ displayName $ employeeNumber $ employeeType $ givenName $ homePhone $ homePostalAddress $ initials $ jpegPhoto $ labeledURI $ mail $ manager $ mobile $ o $ pager $ photo $ roomNumber $ secretary $ uid $ userCertificate $ x500uniqueIdentifier $ preferredLanguage $ userSMIMECertificate $ userPKCS12 $ description $ enabled $ email ) )  
EOF  
  
    ldapmodify -c -Y EXTERNAL -H ldapi:/// -f modify.ldif > /dev/null 2>&1   
  
    if [ $? -eq 0 ]  
    then  
        echo "modify.ldif success"  
    else  
        echo "modify.ldif failed"  
        exit  
    fi  
      
    cat > ./add.ldif <<EOF  
dn: ou=users,dc=openstack,dc=org  
objectClass: top  
objectClass: organizationalUnit  
  
dn: ou=projects,dc=openstack,dc=org  
objectClass: top  
objectClass: organizationalUnit  
  
dn: ou=roles,dc=openstack,dc=org  
objectClass: top  
objectClass: organizationalUnit  
  
dn: ou=groups,dc=openstack,dc=org  
objectClass: top  
objectClass: organizationalUnit  
  
dn: ou=domains,dc=openstack,dc=org  
objectClass: top  
objectClass: organizationalUnit  
EOF  
  
    ldapadd -x -c -D "cn=admin,dc=openstack,dc=org" -w "${LDAP_PASS}" -f add.ldif > /dev/null 2>&1   
      
    if [ $? -eq 0 ]  
    then  
        echo "add.ldif success"  
    else  
        echo "add.ldif failed"  
        exit  
    fi  
      
    rm -rf ./modify.ldif  
    rm -rf ./add.ldif  
  
}  
  
#step3 modify keystone.conf  
function modify_keystone_conf()  
{  
    cat > ./keystone_ldap.conf <<EOF  
url = ldap://localhost  
user = cn=admin,dc=openstack,dc=org  
password = ${LDAP_PASS}  
suffix = dc=openstack,dc=org  
use_dumb_member = True  
allow_subtree_delete = False  
  
user_tree_dn = ou=users,dc=openstack,dc=org  
tenant_tree_dn = ou=projects,dc=openstack,dc=org  
role_tree_dn = ou=roles,dc=openstack,dc=org  
group_tree_dn = ou=groups,dc=openstack,dc=org  
domain_tree_dn = ou=domains,dc=openstack,dc=org  
EOF  
  
    if [ -w "${KEYSTONE_CONF}" ]  
    then  
        sed "s/^\(driver = keystone.identity.backends.\).*\(.Identity\)/\1ldap\2/" -i ${KEYSTONE_CONF}  
        sed "/^\[ldap\]$/r ./keystone_ldap.conf" -i ${KEYSTONE_CONF}  
    else  
        echo "modify keystone.conf failed"  
        exit  
    fi  
      
    rm -rf ./keystone_ldap.conf  
      
    stop keystone && start keystone > /dev/null 2>&1  
    if [ $? -eq 0 ]  
    then  
        echo "restart keystone success"  
    else  
        echo "restart keystone failed"  
        exit  
    fi  
    sleep 2  
}  
  
#step4 init keystone  
function init_keystone()  
{  
        cat > ./keystone_basic.sh <<EOF  
#!/bin/sh  
#  
# Keystone basic configuration   
  
# Mainly inspired by https://github.com/openstack/keystone/blob/master/tools/sample_data.sh  
  
# Modified by Bilel Msekni / Institut Telecom  
#  
# Support: openstack@lists.launchpad.net  
# License: Apache Software License (ASL) 2.0  
#  
#. /root/novarc  
#HOST_IP=\${MASTER}  
ADMIN_PASSWORD=Galax8800  
SERVICE_PASSWORD=Galax8800  
SERVICE_TENANT_NAME=\${SERVICE_TENANT_NAME:-service}  
  
get_id () {  
    echo \`\$@ | awk '/ id / { print \$4 }'\`  
}  
  
# Tenants  
ADMIN_TENANT=\$(get_id keystone tenant-create --name=admin)  
SERVICE_TENANT=\$(get_id keystone tenant-create --name=\$SERVICE_TENANT_NAME)  
  
  
# Users  
ADMIN_USER=\$(get_id keystone user-create --name=admin --pass="\$ADMIN_PASSWORD" --email=admin@domain.com)  
  
  
# Roles  
ADMIN_ROLE=\$(get_id keystone role-create --name=admin)  
KEYSTONEADMIN_ROLE=\$(get_id keystone role-create --name=KeystoneAdmin)  
KEYSTONESERVICE_ROLE=\$(get_id keystone role-create --name=KeystoneServiceAdmin)  
  
# Add Roles to Users in Tenants  
keystone user-role-add --user-id \$ADMIN_USER --role-id \$ADMIN_ROLE --tenant-id \$ADMIN_TENANT  
keystone user-role-add --user-id \$ADMIN_USER --role-id \$KEYSTONEADMIN_ROLE --tenant-id \$ADMIN_TENANT  
keystone user-role-add --user-id \$ADMIN_USER --role-id \$KEYSTONESERVICE_ROLE --tenant-id \$ADMIN_TENANT  
  
# The Member role is used by Horizon and Swift  
MEMBER_ROLE=\$(get_id keystone role-create --name=Member)  
  
# Configure service users/roles  
NOVA_USER=\$(get_id keystone user-create --name=nova --pass="\$SERVICE_PASSWORD" --email=nova@domain.com)  
keystone user-role-add --tenant-id \$SERVICE_TENANT --user-id \$NOVA_USER --role-id \$ADMIN_ROLE  
  
GLANCE_USER=\$(get_id keystone user-create --name=glance --pass="\$SERVICE_PASSWORD" --email=glance@domain.com)  
keystone user-role-add --tenant-id \$SERVICE_TENANT --user-id \$GLANCE_USER --role-id \$ADMIN_ROLE  
  
QUANTUM_USER=\$(get_id keystone user-create --name=quantum --pass="\$SERVICE_PASSWORD" --email=quantum@domain.com)  
keystone user-role-add --tenant-id \$SERVICE_TENANT --user-id \$QUANTUM_USER --role-id \$ADMIN_ROLE  
  
CINDER_USER=\$(get_id keystone user-create --name=cinder --pass="\$SERVICE_PASSWORD" --email=cinder@domain.com)  
keystone user-role-add --tenant-id \$SERVICE_TENANT --user-id \$CINDER_USER --role-id \$ADMIN_ROLE  
EOF  
      
    bash keystone_basic.sh > /dev/null 2>&1  
    if [ $? -eq 0 ]  
    then  
        echo "init keystone success"  
    else  
        echo "init keystone failed"  
        exit  
    fi  
      
    rm -rf ./keystone_basic.sh  
  
}  
  
#step5 check keystone  
function check_keystone()  
{  
    keystone user-list  
    keystone role-list  
    keystone tenant-list  
}  
  
  
#step6 restart openstack all service  
function restart_openstack_all_service()  
{  
    cd /etc/init.d/; for i in $( ls quantum-* ); do sudo service $i restart; done; cd -;  
    cd /etc/init.d/; for i in $( ls nova-* ); do sudo service $i restart; done; cd -;  
    cd /etc/init.d/; for i in $( ls cinder-* ); do sudo service $i restart; done; cd -;  
    cd /etc/init.d/; for i in $( ls glance-* ); do sudo service $i restart; done; cd -;  
    cd /etc/init.d/; for i in $( ls keystone ); do sudo service $i restart; done; cd -;  
}  
  
  
###################### main #######################  
install_ldap  
check_ldap_login  
modify_ldap_schema  
modify_keystone_conf  
init_keystone  
check_keystone  
restart_openstack_all_service
```

脚本中包括：

- step0 install ldap and setpassword

- step1 check ldap login

- step2 modify ldap schema

- step3 modify keystone.conf

- step4 init keystone

- step5 check keystone

- step6 restartopenstack all service

最后一步重启所有openstack服务的步骤，会重启本节点的所有openstack进程，如果有其他节点需要手动重启

运行自动化脚本

`bash config_keystone_ldap.sh`

默认LDAP密码为`Galax8800`

最后附送ubuntu环境下完全删除LDAP的命令

`apt-get purge slapd`

[1]: https://raw.github.com/kiwik/kiwik.github.io/master/_posts_images/2013-12-14/1.jpg
[2]: https://raw.github.com/kiwik/kiwik.github.io/master/_posts_images/2013-12-14/2.jpg
[3]: https://raw.github.com/kiwik/kiwik.github.io/master/_posts_images/2013-12-14/3.jpg
[4]: https://raw.github.com/kiwik/kiwik.github.io/master/_posts_images/2013-12-14/4.jpg
[5]: https://raw.github.com/kiwik/kiwik.github.io/master/_posts_images/2013-12-14/5.jpg
[6]: https://raw.github.com/kiwik/kiwik.github.io/master/_posts_images/2013-12-14/6.jpg
[7]: https://raw.github.com/kiwik/kiwik.github.io/master/_posts_images/2013-12-14/7.jpg