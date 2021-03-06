---
layout: post
title: "Ansible 开发模块 之【企业微信通知】"
date: "2019-07-03 22:05:49"
category: Ansible
tags:  Ansible module
author: lework
---
* content
{:toc}

## 需求

当ansible执行完任务时，通过企业微信通知到用户。

## 实现

使用`ansible`的`fetch_url`获取企业微信接口的token，然后像发送消息的接口`post`我们准备好的数据。




## module编写

可以直接使用 [github地址](https://github.com/lework/Ansible-dev/blob/master/library/wechat.py)

```
cat /usr/share/ansible/plugins/modules/wechat.py
#!/usr/bin/python
# -*- coding: utf-8 -*-
#
# Copyright: (c) 2019, Lework <lework@yeah.net>
# GNU General Public License v3.0+ (see COPYING or https://www.gnu.org/licenses/gpl-3.0.txt)

from __future__ import absolute_import, division, print_function

__metaclass__ = type

ANSIBLE_METADATA = {'metadata_version': '1.1',
                    'status': ['stableinterface'],
                    'supported_by': 'community'}

DOCUMENTATION = '''
---
module: wechat
version_added: "1.0"
short_description: Send a message to Qiye Wechat.
description:
   - Send a message to Qiye Wechat.
options:
  corpid:
    description:
      - Business id.
    required: true
  secret:
    description:
      - application secret.
    required: true
  agentid:
    description:
      - application id.
    required: true
  touser:
    description:
      - Member ID list (message recipient, multiple recipients separated by '|', up to 1000). 
      - "Special case: Specify @all to send to all members of the enterprise application."
      -  When toparty, touser, totag is empty, the default value is @all.
  toparty:
    description:
      - A list of department IDs. Multiple recipients are separated by ‘|’ and support up to 100. Ignore this parameter when touser is @all
  totag:
    description:
      - A list of tag IDs, separated by ‘|’ and supported by up to 100. Ignore this parameter when touser is @all
  msg:
    description:
      - The message body.
    required: true
author:
- lework (@lework)
'''

EXAMPLES = '''
# Send all users.
- wechat:
    corpid: "123"
    secret: "456"
    agentid: "100001"
    msg: Ansible task finished
    
# Send a user.
- wechat:
    corpid: "123"
    secret: "456"
    agentid: "100001"
    touser: "LeWork"
    msg: Ansible task finished
# Send multiple user.
- wechat:
    corpid: "123"
    secret: "456"
    agentid: "100001"
    touser: "LeWork|Lework1|Lework2"
    msg: Ansible task finished
    
# Send a department.
- wechat:
    corpid: "123"
    secret: "456"
    agentid: "100001"
    toparty: "10"
    msg: Ansible task finished
'''

RETURN = """
msg:
  description: The message you attempted to send
  returned: always
  type: str
  sample: "Ansible task finished"
touser:
  description: send user id
  returned: success
  type: str
  sample: "ZhangSan"
toparty:
  description: send department id
  returned: success
  type: str
  sample: "10"
totag:
  description: send tag id
  returned: success
  type: str
  sample: "dev"
wechat_error:
  description: Error message gotten from Wechat API
  returned: failure
  type: str
  sample: "Bad Request: message text is empty"
"""


# ===========================================
# WeChat module specific support methods.
#

import json
import traceback

from ansible.module_utils.basic import AnsibleModule
from ansible.module_utils._text import to_native
from ansible.module_utils.urls import fetch_url


class WeChat(object):
    def __init__(self, module, corpid, secret, agentid):
        """
        初始化
        :param module:  Ansible module
        :param corpid:  企业ID
        :param secret:  密钥
        :param agentid: 应用id
        """
        self.module = module
        self.url = "https://qyapi.weixin.qq.com"
        self.corpid = corpid
        self.secret = secret
        self.agentid = agentid
        self.token = ''
        self.msg = ''

        if self.module.check_mode:
        # In check mode, exit before actually sending the message
          module.exit_json(changed=False)

        self.access_token()

    def access_token(self):
        """
        获取企业微信的 access_token
        :return:
        """
        url_arg = '/cgi-bin/gettoken?corpid={id}&corpsecret={crt}'.format(
            id=self.corpid, crt=self.secret)
        url = self.url + url_arg
        response,info = fetch_url(self.module, url=url)
        text = response.read()
        try: 
          self.token = json.loads(text)['access_token']
        except:
          raise Exception("Invalid corpid or corpsecret，api result:%s" % text)

    def messages(self, msg, touser, toparty, totag):
        """
        构建发送数据格式
        :param msg: 消息内容
        :param touser: 指定用户id
        :param toparty: 指定部门id
        :param totag: 指定标签id
        :return:
        """
        values = {
            "msgtype": 'text',
            "agentid": self.agentid,
            "text": {'content': msg},
            "safe": 0
        }

        if touser:
            values['touser'] = touser
        if toparty:
            values['toparty'] = toparty
        if toparty:
            values['totag'] = totag

        self.msg = json.dumps(values)

    def send_message(self, msg, touser=None, toparty=None, totag=None):
        """
        发送消息
        :param msg: 消息内容
        :param touser: 指定用户id
        :param toparty:  指定部门id
        :param totag: 指定标签id
        :return:
        """
        self.messages(msg, touser, toparty, totag)

        send_url = '{url}/cgi-bin/message/send?access_token={token}'.format(
            url=self.url, token=self.token)
        response,info = fetch_url(self.module,url=send_url, data=self.msg, method='POST')
        text = json.loads(response.read())
        if text['invaliduser'] != '' :
           raise Exception("invalid user: %s" % text['invaliduser'])

    def get_department_user(self, did):
        """
        获取部门成员列表
        :param did:  部门id
        :return:
        """
        send_url = '{url}/cgi-bin/user/simplelist?access_token={token}&department_id={did}&fetch_child=1'.format(
            url=self.url, token=self.token, did=did
        )
        response = requests.get(url=send_url)
        print(response.json())

    def get_department(self):
        """
        获取部门列表
        :return:
        """
        send_url = '{url}/cgi-bin/department/list?access_token={token}'.format(
            url=self.url, token=self.token
        )
        response = requests.get(url=send_url)
        print(response.json())


def main():
    module = AnsibleModule(
        argument_spec=dict(
            corpid=dict(required=True, type='str', no_log=True),
            secret=dict(required=True, type='str', no_log=True),
            agentid=dict(required=True, type='str'),
            msg=dict(required=True, type='str'),
            touser=dict(type='str'),
            toparty=dict(type='str'),
            totag=dict(type='str'),
        ),
        supports_check_mode=True
    )

    corpid = module.params["corpid"]
    secret = module.params["secret"]
    agentid = module.params["agentid"]
    msg = module.params["msg"]
    touser = module.params["touser"]
    toparty = module.params["toparty"]
    totag = module.params["totag"]

    try:
        if not touser and not toparty and not totag:
            touser = "@all"

        wechat = WeChat(module, corpid, secret, agentid)
        wechat.send_message(msg, touser, toparty, totag)

    except Exception as e:
        module.fail_json(msg="unable to send msg: %s" % msg, wechat_error=to_native(e), exception=traceback.format_exc())

    changed = True
    module.exit_json(changed=changed, touser=touser, toparty=toparty, totag=totag, msg=msg)

if __name__ == '__main__':
    main()
```

将脚本放在`/usr/share/ansible/plugins/modules/` 目录下

## 测试

ansible版本
```
ansible --version
ansible 2.8.1
  config file = /etc/ansible/ansible.cfg
  configured module search path = [u'/root/.ansible/plugins/modules', u'/usr/share/ansible/plugins/modules']
  ansible python module location = /usr/lib/python2.7/site-packages/ansible
  executable location = /usr/bin/ansible
  python version = 2.7.5 (default, Apr  9 2019, 14:30:50) [GCC 4.8.5 20150623 (Red Hat 4.8.5-36)]
```

查看模块帮助文档
```
ansible-doc wechat
```

测试的playbook

```
cat test_wechat.yaml
---
- hosts: localhost
  gather_facts: false
  tasks:
  - name: test1
    debug:
      msg: demo

  - name: notice
    wechat:
      corpid: "111111111111111"
      secret: "222222222222"
      agentid: "123"
      touser: "ZhangSan"
      #toparty: "10"
      msg: Ansible task finished
```

> 其中wehcat的参数要设置成自己的

- `corpid`: 企业id
- `secret`: 应用秘钥
- `agentid`: 应用id
- `touser`: 用户id
- `toparty`: 部门id
- `totag`: 标签id
- `msg`: 发送的消息

参数详细说明，请看[微信开发文档](https://work.weixin.qq.com/api/doc#90000/90135/90665)

执行效果

```
ansible-playbook test_wechat.yaml

PLAY [localhost] ******************************************************************************

TASK [test1] ******************************************************************************
ok: [localhost] => {
    "msg": "demo"
}

TASK [notice] ******************************************************************************
changed: [localhost]

PLAY RECAP ******************************************************************************
localhost                  : ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

企业微信提醒
![wechat-notice.jpg](/assets/images/Ansible/wechat-notice.jpg)