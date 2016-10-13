---
layout: post
title: Vim Cheat Sheet
---
# ELKスタック設定手順
## サーバーのアーキテクチャ
![ELK Architecture](https://assets.digitalocean.com/articles/elk/elk-infrastructure.png)
## A. ELKサーバー設定手順（Centos6環境）
1. Open JDK 8 をインストールする

  ```bash
  sudo yum install java-1.8.0-openjdk
  ```

2. Elasticsearchをインストールする

  * RPMにElasticsearchのGPG公開鍵を追加する

    ```bash
    sudo rpm --import http://packages.elastic.co/GPG-KEY-elasticsearch
    ```

  * Elasticsearchの新しいYUMリポジトリを作成する

    ```bash
    echo '[elasticsearch-2.x]
    name=Elasticsearch repository for 2.x packages
    baseurl=http://packages.elastic.co/elasticsearch/2.x/centos
    gpgcheck=1
    gpgkey=http://packages.elastic.co/GPG-KEY-elasticsearch
    enabled=1
    ' | sudo tee /etc/yum.repos.d/elasticsearch.repo
    ```
  * Elastichsearchをインストールする

    ```bash
    sudo yum -y install elasticsearch
    ```
  * Elastichsearchの設定ファイルを編集する

    ```bash
    sudo vi /etc/elasticsearch/elasticsearch.yml
    ```
    `# network.host: xxxx` をuncommentして、localhostを指定する

    ```bash
    network.host: localhost
    ```
  * Elastichsearchを起動する

    ```bash
    sudo service elasticsearch start

    ```
  * Elasticsearchを自動起動するように設定

    ```bash
    sudo chkconfig elasticsearch on
    ```

3. Kibanaをインストールする

  * Kibanaの新しいYUMリポジトリを作成する


    ```bash
    echo '[kibana-4.4]
    name=Kibana repository for 4.4.x packages
    baseurl=http://packages.elastic.co/kibana/4.4/centos
    gpgcheck=1
    gpgkey=http://packages.elastic.co/GPG-KEY-elasticsearch
    enabled=1
    ' | sudo tee /etc/yum.repos.d/kibana.repo
    ```
  * Kibanaをインストールする

    ```bash
    sudo yum -y install kibana
    ```
  * Kibanaの設定ファイルを編集する

    ```bash
    sudo vi /opt/kibana/config/kibana.yml
    ```
    `server.host: "0.0.0.0"`は`server.host: "localhost"`に編集する

    ```bash
    server.host: "localhost"
    ```
  * Kibanaを起動する

    ```bash
    sudo service kibana start

    ```
  * Kibanaを自動起動するように設定

    ```bash
    sudo chkconfig kibana on
    ```
4. Nginxをインストールする

  * YUMにEPELリポジトリを追加する

    ```bash
    sudo yum -y install epel-release
    ```

  * Nginxとhttpd-toolsをインストールする

    ```bash
    sudo yum -y install nginx httpd-tools
    ```

  * htpasswdで管理ユーザーを作成する

    ```bash
    sudo htpasswd -c /etc/nginx/htpasswd.users admin
    ```
  **Kibanaにアクセスするようにパスワードを入力して、忘れないでください。**
  * KibanaのNginxの設定ファイルを作成する

    ```bash
    sudo vi /etc/nginx/conf.d/kibana.conf
    ```
    下設定を追加していきます。

    ```bash
    server {
        listen 80;

        server_name ELK_SERVER_IP_ADDRESS;

        auth_basic "Restricted Access";
        auth_basic_user_file /etc/nginx/htpasswd.users;

        location / {
            proxy_pass http://localhost:5601;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection 'upgrade';
            proxy_set_header Host $host;
            proxy_cache_bypass $http_upgrade;
        }
    }
    ```

  * Nginxを起動する

    ```bash
    sudo service nginx start

    ```
  * Nginxを自動起動するように設定

    ```bash
    sudo chkconfig nginx on
    ```
  **Nginxが動かない場合SELinuxとFirewallをチェックしてください。**

5. Logstashをインストールする
  * Logstashの新しいYUMリポジトリを作成する


    ```bash
    echo '[logstash-2.2]
    name=logstash repository for 2.2 packages
    baseurl=http://packages.elasticsearch.org/logstash/2.2/centos
    gpgcheck=1
    gpgkey=http://packages.elasticsearch.org/GPG-KEY-elasticsearch
    enabled=1
    ' | sudo tee /etc/yum.repos.d/logstash.repo
    ```
  * Logstashをインストールする

    ```bash
    sudo yum -y install logstash
    ```
6. SSL証明書を作成する
  * opensslの設定を編集する

    ```bash
    sudo vi /etc/pki/tls/openssl.cnf
    ```
    `[ v3_ca ]`の下に設定を追加していきます。

    ```bash
    subjectAltName = IP: ELK_SERVER_IP_ADDRESS
    ```
  * `/etc/pki/tls`でSSL証明書と秘密鍵を作成する

    ```bash
    cd /etc/pki/tls
    sudo openssl req -config /etc/pki/tls/openssl.cnf -x509 -days 3650 -batch -nodes -newkey rsa:2048 -keyout private/logstash-forwarder.key -out certs/logstash-forwarder.crt
    ```
  **logstash-forwarder.crtファイルはあとでクライアントサーバーにアップロードしていきます。**

7. Logstashを設定する

  * Filebeat設定

    ```bash
    sudo vi /etc/logstash/conf.d/02-beats-input.conf
    ```
    下設定を追加していきます。

    ```bash
    input {
      beats {
        port => 5044
        ssl => true
        ssl_certificate => "/etc/pki/tls/certs/logstash-forwarder.crt"
        ssl_key => "/etc/pki/tls/private/logstash-forwarder.key"
      }
    }
    ```
  * ログフィルタを追加する

    例：syslog

    ```bash
    sudo vi /etc/logstash/conf.d/10-syslog-filter.conf
    ```
    下設定を追加していきます。
    ```bash
    filter {
      if [type] == "syslog" {
        grok {
          match => { "message" => "%{SYSLOGTIMESTAMP:syslog_timestamp} %{SYSLOGHOST:syslog_hostname} %{DATA:syslog_program}(?:\[%{POSINT:syslog_pid}\])?: %{GREEDYDATA:syslog_message}" }
          add_field => [ "received_at", "%{@timestamp}" ]
          add_field => [ "received_from", "%{host}" ]
        }
        syslog_pri { }
        date {
          match => [ "syslog_timestamp", "MMM  d HH:mm:ss", "MMM dd HH:mm:ss" ]
        }
      }
    }
    ```
  * Elasticsearch設定

    ```bash
    sudo vi /etc/logstash/conf.d/30-elasticsearch-output.conf
    ```
    下設定を追加していきます。

    ```bash
    output {
      elasticsearch {
        hosts => ["localhost:9200"]
        sniffing => true
        manage_template => false
        index => "%{[@metadata][beat]}-%{+YYYY.MM.dd}"
        document_type => "%{[@metadata][type]}"
      }
    }
    ```
    追加した設定を確認する

    ```bash
    sudo service logstash configtest
    ```
    完成場合は`Configuration OK`を表示される
  * Logstashを起動する

    ```bash
    sudo service logstash start

    ```
  * Logstashを自動起動するように設定

    ```bash
    sudo chkconfig logstash on
    ```

8. サンプルKibanaダッシュボードを指定する

  ```bash
  cd ~
  curl -L -O http://download.elastic.co/beats/dashboards/beats-dashboards-1.3.1.zip
  unzip beats-dashboards-1.3.1.zip
  cd beats-dashboards-1.3.1/
  ./load.sh
  ```
9. ElasticsearchのFilebeatインデックスのテンプレートを指定する

  ```bash
  cd ~
  vi filebeat-index-template.json
  ```
  下設定を追加していきます。

  ```bash
  {
    "mappings": {
      "_default_": {
        "_all": {
          "enabled": true,
          "norms": {
            "enabled": false
          }
        },
        "dynamic_templates": [
          {
            "template1": {
              "mapping": {
                "doc_values": true,
                "ignore_above": 1024,
                "index": "not_analyzed",
                "type": "{dynamic_type}"
              },
              "match": "*"
            }
          }
        ],
        "properties": {
          "@timestamp": {
            "type": "date"
          },
          "message": {
            "type": "string",
            "index": "analyzed"
          },
          "offset": {
            "type": "long",
            "doc_values": "true"
          },
          "geoip"  : {
            "type" : "object",
            "dynamic": true,
            "properties" : {
              "location" : { "type" : "geo_point" }
            }
          }
        }
      }
    },
    "settings": {
      "index.refresh_interval": "5s"
    },
    "template": "filebeat-*"
  }
  ```

  そして、Elasticsearchに追加していきます。

  ```bash
  curl -XPUT 'http://localhost:9200/_template/filebeat?pretty' -d@filebeat-index-template.json

  ```

  完成場合は下のコンテンツが表示される

  ```bash
  {
    "acknowledged" : true
  }
  ```


## B. クライアントサーバー設定手順（APPサーバー）
1. SSL証明書をコピーする

  * クライアントサーバーにELKサーバーの`logstash-forwarder.crt`ファイルをアップロードする
  ELKサーバーで：

    ```bash
    scp /etc/pki/tls/certs/logstash-forwarder.crt user@client_server_ip_address:/tmp
    ```
  クライアントサーバーで：

    ```bash
    sudo mkdir -p /etc/pki/tls/certs
    sudo cp /tmp/logstash-forwarder.crt /etc/pki/tls/certs/
    ```
2. FileBeatをインストールする

  * RPMにElasticsearchのGPG公開鍵を追加する

    ```bash
    sudo rpm --import http://packages.elastic.co/GPG-KEY-elasticsearch
    ```

  * Elasticsearchの新しいYUMリポジトリを作成する

    ```bash
    echo '[beats]
    name=Elastic Beats Repository
    baseurl=https://packages.elastic.co/beats/yum/el/$basearch
    enabled=1
    gpgkey=https://packages.elastic.co/GPG-KEY-elasticsearch
    gpgcheck=1
    ' | sudo tee /etc/yum.repos.d/elastic-beats.repo
    ```

  * Elastichsearchをインストールする

    ```bash
    sudo yum -y install filebeat
    ```
3. FileBeatを設定する

  ```bash
  sudo vi /etc/filebeat/filebeat.yml
  ```
  * 設定ファイル編集

    ```bash
    ...
    paths:
      - /var/log/*.log
    ...
    ```
    は下のコンテンツに編集していきます。

    ```bash
    ...
    paths:
      - /var/log/secure
      - /var/log/messages
    ...
    ```

    そして、

    ```bash
    ...
    # document_type: log
    ...

    ```
    をuncommentして、下のコンテンツに編集していきます。

    ```bash
    ...
    document_type: syslog
    ...

    ```

    `output`部分でelasticsearch部分（`elasticsearch:`から`logstash:`まで）を全部コメントアウトする（使わな為）

    次に、

    `#logstash:`をuncommentする

    `#hosts: ["localhost:5044"]`をuncommentして、localhostは`ELK_SERVER_IP_ADDRESS`に編集していきます。

    `hosts: ["ELK_SERVER_IP_ADDRESS:5044"]`の下に`bulk_max_size: 1024`を追加していきます。

    次に、

    `tls`部分で`#certificate_authorities`をuncommentして、下のコンテンツに編集していきます。

    ```bash
    certificate_authorities: ["/etc/pki/tls/certs/logstash-forwarder.crt"]
    ```

  * Filebeatを起動する

    ```bash
    sudo service filebeat start

    ```
  * Filebeatを自動起動するように設定

    ```bash
    sudo chkconfig filebeat on
    ```

4. 設定確認

  * ELKサーバーで

    ```bash
    curl -XGET 'http://localhost:9200/filebeat-*/_search?pretty'
    ```
    を実行していきます。完成場合は

    ```bash
    ...
    {
          "_index" : "filebeat-2016.01.29",
          "_type" : "log",
          "_id" : "AVKO98yuaHvsHQLa53HE",
          "_score" : 1.0,
          "_source":{"message":"Feb  3 14:34:00 rails sshd[963]: Server listening on :: port 22.","@version":"1","@timestamp":"2016-01-29T19:59:09.145Z","beat":{"hostname":"topbeat-u-03","name":"topbeat-u-03"},"count":1,"fields":null,"input_type":"log","offset":70,"source":"/var/log/auth.log","type":"log","host":"topbeat-u-03"}
        }
    ...
    ```
    を表示される

  * ブラウザでELK_SERVER_IP_ADDRESSをアクセスする

  ![](https://assets.digitalocean.com/articles/elk/1-filebeat-index.gif)

  Settings->Indeciesで`[filebeat]-YYY.MM.DD`テンプレートを指定してください。
  今からKibanaでログを検索できます。