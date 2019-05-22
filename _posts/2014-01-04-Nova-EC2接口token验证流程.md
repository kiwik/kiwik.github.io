---
layout: post
category : OpenStack
tagline: "regexisart"
tags : [OpenStack, nova, keystone, EC2, token]
---
{% include JB/setup %}

**如需转载，请标明原文出处以及作者**

*陈锐 RuiChen @kiwik*

*2014/1/4 13:56:10 *

----------

一直比较好奇Nova的EC2 API和Keystone EC2扩展都是怎么运作的，两者之间有什么关系，这段时间抽空琢磨了一下。

## AWS API Client

先要准备一个AWS EC2的Client工具，当前在Ubuntu上有两个分类的Client，euca2ools和ec2-api-tools、ec2-ami-tools，我两个都用了一下，euca2ools可以直接使用，ec2-api-tool和ec2-ami-tools安装成功，但是执行命令的时候，报证书错误不能使用，错误原因和本文关系不大，本文以euca2ools为例。

`apt-get install -y euca2ools`

## Keystone

使用euca2ools之前，首先要通过keystone创建EC2的acces\_key和secret\_key，直接使用命令`keystone ec2-credentials-create`为某个project下的某个user创建一组access\_key和secret\_key，以备后面使用

![](https://raw.github.com/kiwik/kiwik.github.io/master/_posts_images/2014-01-04/2.png)

## Horizon

突然就拐到了Horizon，是因为当前的Horizon也有对于EC2 tools的支持，当我们用某一个用户登录Horizon的时候，左侧的“项目”tab中有一个“访问 & 安全”的链接，点击后页面右上角就会有一个”下载EC2凭证“的按钮，点击之后会从浏览器下载一个命名为${tenant_name}-x509.zip的zip包。使用哪个用户登录就会使用这个用户已经在keystone创建的access\_key和secret\_key

![](https://raw.github.com/kiwik/kiwik.github.io/master/_posts_images/2014-01-04/1.png)

需要注意的是，想要下载EC2凭证，nova-cert服务必须启动，因为需要nova-cert生成对应tenant的cert.pem和pk.pem

解开zip包之后，里面就包括cacert.pem、pk.pem、cert.pem三个pem文件和一个shell脚本ec2rc.sh，以下就是ec2rc.sh的内容，使用euca2ools之前，先要执行一下 `source ec2rc.sh` ，三个pem文件是EC2 Client对于image、volume压缩打包加密时用的。

{% highlight bash %}

#!/bin/bash

NOVARC=$(readlink -f "${BASH_SOURCE:-${0}}" 2>/dev/null) || NOVARC=$(python -c 'import os,sys; print os.path.abspath(os.path.realpath(sys.argv[1]))' "${BASH_SOURCE:-${0}}")
NOVA_KEY_DIR=${NOVARC%/*}
export EC2_ACCESS_KEY=14d40f23e54148579ea559c07bfaa42a
export EC2_SECRET_KEY=921a3445e0c2485d83458251d1803219
export EC2_URL=http://172.25.16.1:8773/services/Cloud
export EC2_USER_ID=42 # nova does not use user id, but bundling requires it
export EC2_PRIVATE_KEY=${NOVA_KEY_DIR}/pk.pem
export EC2_CERT=${NOVA_KEY_DIR}/cert.pem
export NOVA_CERT=${NOVA_KEY_DIR}/cacert.pem
export EUCALYPTUS_CERT=${NOVA_CERT} # euca-bundle-image seems to require this set

alias ec2-bundle-image="ec2-bundle-image --cert ${EC2_CERT} --privatekey ${EC2_PRIVATE_KEY} --user 42 --ec2cert ${NOVA_CERT}"
alias ec2-upload-bundle="ec2-upload-bundle -a ${EC2_ACCESS_KEY} -s ${EC2_SECRET_KEY} --url ${S3_URL} --ec2cert ${NOVA_CERT}"

{% endhighlight %}

其中最有用的就是以下三个环境变量，EC2\_URL就是指向nova的ec2-api，EC2_ACCESS_KEY和EC2_SECRET_KEY就是，通过keystone ec2-credentials-create创建的某project的某个用户的access\_key和secret\_key

> export EC2_ACCESS_KEY=14d40f23e54148579ea559c07bfaa42a
>
> export EC2_SECRET_KEY=921a3445e0c2485d83458251d1803219
>
> export EC2_URL=http://172.25.16.1:8773/services/Cloud

## Nova EC2 API

如果在nova.conf配置ec2-api为enabled，那么nova-api进程启动的时候就会为ec2-api单独fork一个子进程，在8773端口监听，处理EC2类型的请求。

看一下nova的api-paste.ini里面专门有一段是关于EC2 API的配置的

{% highlight ini %}

[composite:ec2]
use = egg:Paste#urlmap
/services/Cloud: ec2cloud

[composite:ec2cloud]
use = call:nova.api.auth:pipeline_factory
noauth = ec2faultwrap logrequest ec2noauth cloudrequest validator ec2executor
keystone = ec2faultwrap logrequest ec2keystoneauth cloudrequest validator ec2executor

...

[filter:ec2keystoneauth]
paste.filter_factory = nova.api.ec2:EC2KeystoneAuth.factory

...

[filter:cloudrequest]
controller = nova.api.ec2.cloud.CloudController
paste.filter_factory = nova.api.ec2:Requestify.factory

{% endhighlight %}

其中和本文相关的就是 *ec2keystoneauth* 和 *cloudrequest* 两个filter


## ec2keystoneauth

ec2keystoneauth就是用来兼容EC2类型的鉴权请求的，在这个filter中Nova将request请求的参数params，例如：Signature签名， AWSAccessKeyId接入key等，都通过 *nova.conf* 中配置的keystone_ec2_url发给keystone处理，keystone_ec2_url默认为http://localhost:5000/v2.0/ec2tokens

keystone接受的请求就是类似这样的

{% highlight bash %}

{
    'access': '14d40f23e54148579ea559c07bfaa42a',
    'host': '172.25.16.1: 8773',
    'verb': 'POST',
    'params': {
        'SignatureVersion': '2',
        'AWSAccessKeyId': '14d40f23e54148579ea559c07bfaa42a',
        'Timestamp': '2014-01-03T01: 09: 36Z',
        'SignatureMethod': 'HmacSHA256',
        'Version': '2010-08-31',
        'Action': 'DescribeAvailabilityZones'
    },
    'signature': '3zuQL78asBD+SaknEQ0BoJS6ABflN9KpR7ShaTiQK8I=',
    'path': '/services/Cloud/'
}

{% endhighlight %}

keystone会在Ec2Controller中的authenticate方法处理这个POST请求，通过access\_key查询证书，得到secret\_key，然后根据request参数中SignatureVersion，使用不同的签名生成方法结合secret\_key，验证请求中的Signature是否合法，以保证请求不被篡改。验证成功之后，就根据证书的user\_id和project等信息生成一个v2 token。

keystone返回的就是一个与环境变量$EC2\_ACCESS\_KEY和$EC2\_SECRET\_KEY对应的用户的token，nova然后将token中的user\_id，tenant\_id，roles等信息保存在nova的RequestContext中，然后继续处理，后面的处理就和OpenStack的token一致了。

## cloudrequest

会根据时间戳验证请求是否过期，同时根据请求参数中的Action字段确定执行CloudController中将要执行的方法，将Action和CloudController封装在APIRequest中保存在req.environ里，继续向后传递等到ec2executor中执行。

## EC2消息

以下就是一个查询az的euca2ools命令行的debug输出，大家可以仔细看一下euca2ools是如何拼装EC2消息的。

{% highlight bash %}

root@C16-RH2285-01-openstack-controlor-H:/home/chenrui/test-x509# source ec2rc.sh 
root@C16-RH2285-01-openstack-controlor-H:/home/chenrui/test-x509# env | grep EC2
EC2_SECRET_KEY=921a3445e0c2485d83458251d1803219
EC2_USER_ID=42
EC2_URL=http://172.25.16.1:8773/services/Cloud
EC2_ACCESS_KEY=14d40f23e54148579ea559c07bfaa42a
EC2_PRIVATE_KEY=/home/chenrui/test-x509/pk.pem
EC2_CERT=/home/chenrui/test-x509/cert.pem

root@C16-RH2285-01-openstack-controlor-H:/home/chenrui/test-x509# euca-describe-availability-zones --debug
2013-12-31 09:34:25,122 euca2ools [DEBUG]:Using access key provided by client.
2013-12-31 09:34:25,122 euca2ools [DEBUG]:Using secret key provided by client.
2013-12-31 09:34:25,122 euca2ools [DEBUG]:Method: POST
2013-12-31 09:34:25,122 euca2ools [DEBUG]:Path: /services/Cloud/
2013-12-31 09:34:25,122 euca2ools [DEBUG]:Data: 
2013-12-31 09:34:25,122 euca2ools [DEBUG]:Headers: {}
2013-12-31 09:34:25,123 euca2ools [DEBUG]:Host: 172.25.16.1:8773
2013-12-31 09:34:25,123 euca2ools [DEBUG]:Params: {'Action': 'DescribeAvailabilityZones', 'Version': '2010-08-31'}
2013-12-31 09:34:25,123 euca2ools [DEBUG]:establishing HTTP connection: kwargs={'timeout': 70}
2013-12-31 09:34:25,123 euca2ools [DEBUG]:Token: None
2013-12-31 09:34:25,123 euca2ools [DEBUG]:using _calc_signature_2
2013-12-31 09:34:25,123 euca2ools [DEBUG]:query string: AWSAccessKeyId=14d40f23e54148579ea559c07bfaa42a&Action=DescribeAvailabilityZones&SignatureMethod=HmacSHA256&SignatureVersion=2&Timestamp=2013-12-31T14%3A34%3A25Z&Version=2010-08-31
2013-12-31 09:34:25,124 euca2ools [DEBUG]:string_to_sign: POST
172.25.16.1:8773
/services/Cloud/
AWSAccessKeyId=14d40f23e54148579ea559c07bfaa42a&Action=DescribeAvailabilityZones&SignatureMethod=HmacSHA256&SignatureVersion=2&Timestamp=2013-12-31T14%3A34%3A25Z&Version=2010-08-31
2013-12-31 09:34:25,124 euca2ools [DEBUG]:len(b64)=44
2013-12-31 09:34:25,124 euca2ools [DEBUG]:base64 encoded digest: doK1P+ImDAr9YXpWbH6pFHNJtBbNrWMB9lrKqawIkMg=
2013-12-31 09:34:25,124 euca2ools [DEBUG]:query_string: AWSAccessKeyId=14d40f23e54148579ea559c07bfaa42a&Action=DescribeAvailabilityZones&SignatureMethod=HmacSHA256&SignatureVersion=2&Timestamp=2013-12-31T14%3A34%3A25Z&Version=2010-08-31 Signature: doK1P+ImDAr9YXpWbH6pFHNJtBbNrWMB9lrKqawIkMg=
send: 'POST /services/Cloud/ HTTP/1.1\r\nHost: 172.25.16.1:8773\r\nAccept-Encoding: identity\r\nContent-Length: 239\r\nContent-Type: application/x-www-form-urlencoded; charset=UTF-8\r\nUser-Agent: Boto/2.9.6 (linux2)\r\n\r\nAWSAccessKeyId=14d40f23e54148579ea559c07bfaa42a&Action=DescribeAvailabilityZones&SignatureMethod=HmacSHA256&SignatureVersion=2&Timestamp=2013-12-31T14%3A34%3A25Z&Version=2010-08-31&Signature=doK1P%2BImDAr9YXpWbH6pFHNJtBbNrWMB9lrKqawIkMg%3D'
reply: 'HTTP/1.1 200 OK\r\n'
header: Content-Type: text/xml
header: Content-Length: 526
header: Date: Tue, 31 Dec 2013 14:34:25 GMT
2013-12-31 09:34:25,213 euca2ools [DEBUG]:<DescribeAvailabilityZonesResponse xmlns="http://ec2.amazonaws.com/doc/2010-08-31/">
  <requestId>req-d5cebfd7-d6f2-47f4-b3bc-ae9794de3479</requestId>
  <availabilityZoneInfo>
    <item>
      <zoneState>available</zoneState>
      <zoneName>test</zoneName>
    </item>
    <item>
      <zoneState>available</zoneState>
      <zoneName>nova</zoneName>
    </item>
    <item>
      <zoneState>available</zoneState>
      <zoneName>aztest001</zoneName>
    </item>
  </availabilityZoneInfo>
</DescribeAvailabilityZonesResponse>

AVAILABILITYZONE        test    available
AVAILABILITYZONE        nova    available
AVAILABILITYZONE        aztest001       available

{% endhighlight %}

## 参考链接

[http://wherenow.org/openstack-nova-ec2-api-flow/](http://wherenow.org/openstack-nova-ec2-api-flow/)

[https://computing.seas.harvard.edu/display/CLOUD/EC2+for+Openstack](https://computing.seas.harvard.edu/display/CLOUD/EC2+for+Openstack)
