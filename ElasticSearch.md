
* ## 搜索 
    * ### <font color="red">ES前面是需要有反向代理的，因为这样会安全许多。不要让elasticSearch直接面对最前端，要在es前面架设2台，起码架设2台能做反向代理的WEB服务器。不是说你有F5或者其他负载均衡设备就可以了，当然了，除非你乐意在防火墙那层面对ES做专门的措施，比如在防火墙那里动态对访问ES的所有请求的HEADER都加上一个认证参数。否则的话，请一定在ES集群前架设两台反向代理用的WEB服务器。省钱的方案是弄两台装NGINX（或者衍生品）的服务器，每台除了NGINX还要配置LVS做好浮动IP（如果你有F5这类负载均衡，可以不用LVS，而由负载均衡代替浮动IP），记得在NGINX上配置访问ES的用户名密码。后面也会介绍一点KIBANA，这样的话，你就可以为ES增加不同的用户、密码以及相应的权限，给搜索用的用户权限一定要尽量低（注意安全补丁）</font>
        > 通过监控生成的静态文件目录，或者在静态化期间将待搜索内容（标题，内容等）及相关联的资源（如外网访问的链接）存入ElasticSearch。ElasticSearch本身提供二进制包，可在 https://www.elastic.co/cn/ 进行下载。

    * ### 安装
        ####前置

        ```bash
        #如果是centos这类系统，建议先做以下事情(ubuntu也差不多):
        vim etc/sysctrl.conf
          fs.file-max = 1000000
          vm.max_map_count=262144
          vm.swappiness = 1
        vim /etc/security/limits.conf
          * soft nofile 65536
          * hard nofile 131072
        vi /etc/security/limits.d/90-nproc.conf
          *          soft    nproc     2048        

        #配置java1.8或者写到profile里source
        export JAVA_HOME=/data/jdk1.8.0_151
        export PATH=$JAVA_HOME/bin:$PATH
        export CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar

        #新建用户elk 安装用root 运行用elk
        groupadd elk
        useradd elk -g elk
        mkdir -p /data/elk
        ```

        ####安装x-pack
        ```bash
        #安装elasticsearch logstash kibana        

        #安装x-pack
        bin/elasticsearch-plugin install file:///home/zzuser/elk/x-pack-5.6.3.zip
        bin/logstash-plugin install file:///home/zzuser/elk/x-pack-5.6.3.zip
        bin/kibana-plugin install file:///home/zzuser/elk/x-pack-5.6.3.zip
        #破解x-pack
        #启动elasticsearch再停止后，替换x-pack-5.6.3.jar
        #https://license.elastic.co/registration申请基础许可license.json邮件
        #修改license文件: "type":"platinum" "expiry_date_in_millis":146128000000 一年后的时间戳
        #更新license文件: curl -XPUT -u elastic:changeme 'http://127.0.0.1:9200/_xpack/license' -d @license.json
        ```

        ####安装其他plugin
        ```bash
        #elasticsearch-plugin安装分词
        bin/elasticsearch-plugin install file:///home/zzuser/elk/ik.zip
        #用elk用户启动
        vi elasticsearch.yml
        bin/elasticsearch -d -v

        #logstash-plugin安装http input
        bin/logstash-plugin install logstash-input-http
        #其他安装方法
        #vi Gemfile >> gem "logstash-input-http", :path => "/root/logstash-input-http-master"
        #bin/logstash-plugin install --no-verify
        #yum install ruby
        #yum install rubygems
        #unzip master
        #gem build xxx.gemspec 
        #logstash-plugin install xxx.gem
        #用elk用户启动 并指定配置文件
        vi logstash.conf
        bin/logstash -f config/logstash.conf

        #kibana
        bin/kibana
        ```


    * ### 配置
        > 下面给出测试时的配置（个人喜好三主三数据这样的）        

   * #### ElasticSearch配置
        ```nginx
        #集群名
        cluster.name: search
        #节点名称
        node.name: node1
        #是否为主节点
        node.master: true
        #是否为数据节点
        node.data: false
        #绑定地址
        network.host: 172.16.100.31
        #该集群内的机器有男鞋
        discovery.zen.ping.unicast.hosts: ["172.16.100.30", "172.16.130.31", "172.16.130.32"]
        #一个节点需要看到的具有master节点资格的最小数量，然后才能在集群中做操作。官方的推荐值是(N/2)+1
        discovery.zen.minimum_master_nodes: 2
        #设置集群中N个节点启动后进行数据恢复
        gateway.recover_after_nodes: 3
        #其他的可在网上进行查找
        bootstrap.memory_lock: false
        bootstrap.system_call_filter: false
        ```
   * #### logstash配置
        ```nginx
        input {
          http{
              #输入端所监听的地址、端口
              host => "0.0.0.0"
              port => 8111
              #内容解码类型、内容字符集
              codec => json {
                charset => ["UTF-8"]
              }
              #线程数
              threads => 4
              #用户名密码
              user => tran
              password => zerotest
              #是否使用ssl
              ssl => false
          }
        }
        output {
          #可同时向标准、es输出
          # 标准输出解码方式
          stdout {  codec => rubydebug }
          elasticsearch {
                 #es所在的地址以及端口
                 hosts => ["http://127.0.0.1:9200"]
                 #希望es做什么动作（这个字段在后面写到了程序里，可以由程序来控制增加、更新、删除）
                 action => "%{action}"
                 #用户名密码
                 user => elastic
                 password => changeme
                 #文档唯一ID
                 document_id => "%{hashid}"
                 #此文档要使用哪个索引
                 index => "%{indexname}"
                 #此文档要使用哪个文档类型（可以理解document_type是表、index是库。。。不要完全照搬rl类数据库的原始概念）
                 document_type => mixdoc
                 doc_as_upsert => true
          }
        }

        ```

        > 下一步动作之前开始安装ik，x-pack等插件，ik是中文分词，需要安装到es下，x-pack是安全、监控等功能的官方插件（收费），是选择性使用的东西，用了后有漂亮的监控界面，不用也不影响系统正常使用，就是监控相应的情况麻烦。当然，有那啥。。。开始建立索引和文档类型(在装好ik的情况下,另外es系统缺省用户名密码为elastic:changeme)，以索引名somename举例

        ```bash
        #先建立somename_v1这个索引,参数都很好理解就不多说了
        curl -u elastic:changeme -XPUT 'http://172.16.100.30:9200/somename_v1' -d'
        {
          "settings": {
            "number_of_shards": 10,
            "number_of_replicas": 1,
            "index": {
              "analysis": {
                "analyzer": {
                  "by_smart": {
                    "type": "custom",
                    "tokenizer": "ik_smart",
                    "filter": ["by_tfr","by_sfr"],
                    "char_filter": ["by_cfr"]
                  },
                  "by_max_word": {
                    "type": "custom",
                    "tokenizer": "ik_max_word",
                    "filter": ["by_tfr","by_sfr"],
                    "char_filter": ["by_cfr"]
                  }
                },
                "filter": {
                  "by_tfr": {
                    "type": "stop",
                    "stopwords": [" "]
                  },
                  "by_sfr": {
                    "type": "synonym",
                    "synonyms_path": "analysis/synonyms.txt"
                  }
                },
                "char_filter": {
                  "by_cfr": {
                    "type": "mapping",
                    "mappings": ["| => |"]
                  }
                }
              }
            }    
          }
        }'        
        ```

        > 建立文档映射（document_type类似表及其内部字段）

        ```
        curl  -u elastic:changeme  -XPUT 'http://172.16.100.30:9200/somename_v1/_mapping/mixdoc' -d'
        {
          "properties": {
            "hashid" : {
              "type": "keyword"
            },
            "title": {
              "type": "text",
              "index": true,
              "analyzer": "by_max_word",
              "search_analyzer": "by_smart"
            },
            "content": {
              "type": "text",
              "index": true,
              "analyzer": "by_max_word",
              "search_analyzer": "by_smart"
            },
            "keys": {
              "type": "keyword"
            },
            "date": {
              "type": "date"
            },
            "description": {
              "type": "text"
            },
            "pics": {
              "type": "text"
            },
            "link": {
              "type": "text"
            },
            "author": {
              "type": "keyword"
            },
            "mark_int": {
              "type": "long"
            },
            "mark_text": {
              "type": "text"
            },    
            "area_location": {
              "type": "geo_point"
            },
            "area_shape": {
              "type": "geo_shape"
            },
            "other_backup1": {
              "type": "text",
              "index": false
            },
            "other_backup2": {
              "type": "text",
              "index": false
            }          
          }
        }'        
        ```

        > 通过别名方式建立真正的somename这个doucment_type(这么做的好处是因为es不支持为document_type增加字段，如果增加就会全量索引，为了避免更迭出现的问题，用别名的方式建立最好，这样可以做成类似链表的接口，改变指针即可)

        ```bash
        curl -u elastic:changeme -XPOST 172.16.100.30:9200/_aliases -d '
        {
            "actions": [
                { "add": {
                    "alias": "somename",
                    "index": "somename_v1"
                }}
            ]
        }'        
        ```

        > 静态化内容存入ES的方式有很多，如果是涉及到已经成型的发布系统，而发布系统本身没有对外提供相应的自动化数据接口，那么就我个人的习惯会根据历史数据量的多少、每天更新量的多少、本项目是否与其他项目混用同一级别的发布目录来进行不同方案的选择。因为本项目实质上内容不会太多，所以直接采用了监控目录的方法，方法很简单，使用linux内核的inotify机制（对系统内核有一点点要求，这个请自行查找），采用类似下面的python代码即可，届时只需稍微提高代码的容错性并实现故障的情况下自动重启（<font color="red">个人建议supervisor</font>），并要将打印部分的代码修改为连接kafka或者存入数据库，另外再写一个消费这些信息的程序将数据写入logstash或者直接写入elasticSearch即可，这两个部分的示意代码如下(py3比较好。至于py2.。。。):

        >  监控目录的示例程序是:

        ```python
        import os
        from  pyinotify import * # WatchManager, Notifier, ProcessEvent,IN_DELETE, IN_CREATE,IN_MODIFY,IN_CLOSE_WRITE,IN_CLOSE_NOWRITE
        class EventHandler(ProcessEvent):
            """事件处理"""
            max_queued_events.value = 99999
            def process_IN_CREATE(self, event):
                findIndex = event.name.find('index')
                findHtm = event.name.find('htm')
                if findIndex >= 0 and findHtm >= 0:
                    print("Create file: %s "  %   os.path.join(event.path,event.name))

            def process_IN_DELETE(self, event):
                findIndex = event.name.find('index')
                findHtm = event.name.find('htm')
                if findIndex >= 0 and findHtm >= 0:
                    print("Delete file: %s "  %   os.path.join(event.path,event.name))

            def process_IN_MODIFY(self, event):
                findIndex = event.name.find('index')
                findHtm = event.name.find('htm')
                if findIndex >= 0 and findHtm >= 0:
                    print("Modify file: %s "  %   os.path.join(event.path,event.name))

            def process_IN_CLOSE_WRITE(self, event):
                findIndex = event.name.find('index')
                findHtm = event.name.find('htm')
                if findIndex >= 0 and findHtm >= 0:
                    print("CLOSE WRITE file: %s "  %   os.path.join(event.path,event.name))

            def process_IN_NOMODIFY(self, event):
                findIndex = event.name.find('index')
                findHtm = event.name.find('htm')
                if findIndex >= 0 and findHtm >= 0:
                    print("CLOSE NOWRITE: %s "  %   os.path.join(event.path,event.name))

            def process_IN_Q_OVERFLOW(self, event):
                print('-_-',max_queued_events.value)
                max_queued_events.value *= 3

        def FSMonitor(path='.'):
            wm = WatchManager()
            mask = IN_DELETE | IN_CLOSE_WRITE | IN_CLOSE_NOWRITE
            notifier = Notifier(wm, EventHandler())
            wm.add_watch(path, mask,rec=True)
            print('now starting monitor %s'%(path))
            while True:
                try:
                    notifier.process_events()
                    if notifier.check_events():
                        notifier.read_events()
                except KeyboardInterrupt:
                    notifier.stop()
                    break

        if __name__ == "__main__":
            FSMonitor('/data/wwwroot/')        
        ```

        >  向logstash传递数据的示例程序(此示例程序用于搜索某个目录下的所有html文件，并解析出其标题、内容等项，并传递给logstash):

        ```python
        from bs4 import BeautifulSoup
        import json
        import os
        import os.path
        import hashlib
        import urllib3
        http = urllib3.PoolManager()
        logstash = 'http://172.16.130.52:8080'
        rootdir="/data/wwwroot/"
        _counter = 0
        for parent,dirnames,filenames in os.walk(rootdir):
            for filename in filenames:
                if filename.find('.html') < 0:
                    continue
                completeFileName = os.path.join(parent,filename)
                hash = hashlib.sha256()
                hash.update(completeFileName.encode('utf-8'))
                #为每一个文件生成唯一的ID
                myHash = hash.hexdigest()
                #打开文件夹下的一个文件
                with open(completeFileName,'rb') as fh:
                    soup = BeautifulSoup(fh,'html5lib')
                fh.close()
                #下面基本都是解包、搜索对应特征标签，提出相关内容。使用者需要根据自己页面的结构自行修改
                [m.extract() for m in soup.find_all('style')]
                titleEle = soup.find(name='h1')
                if titleEle is None:
                    continue
                contentEle = soup.find(name='div',class_='TRS_Editor')
                if contentEle is None:
                    continue
                txtContent = contentEle.get_text().replace('\n','')
                titleContent = titleEle.get_text()
                urlPrefix = 'http://www.somename.com/'
                url = completeFileName.replace(rootdir, urlPrefix)
                #这里因为是全量修改，所以'action'对应的永远是index，index一般是出现在要求重新索引（更新），新增内容时候所传递的action。指定hashid是处于要兼容更新、新增两种操作，对于update，有hashid能用于找到数据，对于insert new record可视作人工添加主索引数据，title是标题，content是内容，url是指在外网访问的数据的地址
                normal = json.dumps({'action':'index', 'hashid': myHash, 'title': titleContent, 'content':txtContent, 'url':  url}, ensure_ascii=False)
                result = http.request('POST', logstash, body=normal.encode('utf-8'), headers={'Content-Type': 'application/json'})
                if result.status == 200:
                    #print(normal)
                    _counter += 1
                    print("success:", myHash)
                    if _counter % 10000 == 0:
                      print("_counter", _counter)
                else:
                    print("fail:", myHash, ":", result.status,":", completeFileName)

        ```

> 下面需要介绍一下nginx配置访问ES的密码这部分，同时要注意有HOST问题