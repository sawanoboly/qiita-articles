<!-- too_old -->
> **この記事は最終更新から1年以上経過しています。** 気をつけてね。

ohaiコマンドで出力されるJsonは、Ohaiプラグイン達による`automatic attributes`です。

これはこれで良いですが、ChefではCookbookのなかでどんどんAttributesを追加していくように作られているため、`ChefServer`がない環境では最終的に`node`が持つ`attributes`はかなり拡張されていますが意外と知ることができません。

`node`はMashオブジェクト(Hash拡張)なので、折角だからダンプして確認してみます。


## レシピ

```ruby:node_dump/recipes/default.rb
require 'json'

file "/tmp/dna.json" do
  content JSON.pretty_generate(node)
end
```

これをrun_listの最後にでも入れておけば、最終的にnodeがもつ`attributes`が確認できてdebug等に使えます。


## ついでにAWS OpsWorksで追加されるAttributesを確認する

クラスメソッドさんが**AWS OpsWorks**について書かれています。
[AWS OpsWorksで持っている値をCustom Chef Recipeの中で使いたい！ ｜ Developers.IO:](http://dev.classmethod.jp/cloud/aws/opsworks-deploy-variable/)

> 本当はApplicationの値だけでなくて、LayerとかStackで決めた値も取れるかなと期待していたのですが、今回見つける事ができませんでした。
もしかしたら他のところにあるかもしれないので、見つけ次第追記します。 

カスタムレシピとして`configure`の`custom recipe`リストに先ほどのレシピを追加しました。

node[:opsworks]以下にstackの情報がありますね。


```json:/tmp/dna.json
{
    "json_class": "Chef::Node",
    "chef_type": "node",
    "override": {
        "etc": {
            "passwd": {
              }
          },
        "deploy": {
          }
      },
    "automatic": {
"--snip--": null
      },
    "run_list": [
      "recipe[opsworks_custom_cookbooks::load]",
      "recipe[opsworks_custom_cookbooks::execute]"
    ],
    "normal": {
        "tags": [

        ],
        "opsworks_bundler": {
            "version": "1.0.10",
            "manage_package": null
          },
        "opsworks_test_suite_loader": {
            "minitest_chef_handler": {
                "version": "0.6.7"
              }
          },
        "opsworks_rubygems": {
            "version": "1.8.24"
          },
        "etc": {
            "passwd": {
              }
          },
        "packages": {
            "dist_only": false
          },
        "opsworks_custom_cookbooks": {
            "scm": {
                "repository": "git://github.com/higanworks/opsworks-cookbooks.git",
                "type": "git",
                "revision": null,
                "user": null,
                "ssh_key": null,
                "password": null
              },
            "enabled": true,
            "recipes": [
              "opsworks_ganglia::configure-client",
              "ssh_users",
              "agent_version",
              "opsworks_stack_state_sync",
              "node_dump::default",
              "test_suite",
              "opsworks_cleanup"
            ]
          },
        "nginx": {
            "binary": "/usr/sbin/nginx",
            "log_dir": "/var/log/nginx",
            "user": "www-data",
            "dir": "/etc/nginx"
          },
        "runit": {
            "service_dir": "/etc/service",
            "chpst_bin": "/usr/bin/chpst",
            "sv_dir": "/etc/sv",
            "sv_bin": "/usr/bin/sv"
          },
        "deploy": {
            "wp": {
                "deploy_to": "/srv/www/wp",
                "mounted_at": null,
                "document_root": null,
                "symlinks": {
                  },
                "database": {
                    "database": "wp",
                    "username": "root",
                    "reconnect": true,
                    "host": null,
                    "password": "wiuhfzhstl"
                  },
                "scm": {
                    "repository": "git://github.com/WordPress/WordPress.git",
                    "revision": "3.5.1",
                    "user": null,
                    "scm_type": "git",
                    "ssh_key": null,
                    "password": null
                  },
                "restart_command": "echo 'restarting app'",
                "migrate": false,
                "sleep_before_restart": 0,
                "ssl_certificate_key": null,
                "group": "www-data",
                "user": "www-data",
                "domains": [
                  "wp"
                ],
                "ssl_support": false,
                "ssl_certificate": null,
                "application": "wp",
                "application_type": "php",
                "memcached": {
                    "host": null,
                    "port": 11211
                  },
                "rails_env": null,
                "ssl_certificate_ca": null,
                "symlink_before_migrate": {
                    "config/opsworks.php": "opsworks.php"
                  },
                "auto_bundle_on_deploy": true
              },
            "c5": {
                "deploy_to": "/srv/www/c5",
                "mounted_at": null,
                "document_root": "web",
                "symlinks": {
                  },
                "database": {
                    "database": "c5",
                    "username": "root",
                    "reconnect": true,
                    "host": null,
                    "password": "wiuhfzhstl"
                  },
                "scm": {
                    "repository": "git://github.com/concrete5/concrete5.git",
                    "revision": "5.6.1.2",
                    "user": null,
                    "scm_type": "git",
                    "ssh_key": null,
                    "password": null
                  },
                "restart_command": "echo 'restarting app'",
                "migrate": false,
                "sleep_before_restart": 0,
                "ssl_certificate_key": null,
                "group": "www-data",
                "user": "www-data",
                "domains": [
                  "c5"
                ],
                "ssl_support": false,
                "ssl_certificate": null,
                "application": "c5",
                "application_type": "php",
                "memcached": {
                    "host": null,
                    "port": 11211
                  },
                "rails_env": null,
                "ssl_certificate_ca": null,
                "symlink_before_migrate": {
                    "config/opsworks.php": "opsworks.php"
                  },
                "auto_bundle_on_deploy": true
              }
          },
        "opsworks": {
            "activity": "configure",
            "instance": {
                "instance_type": "m1.small",
                "availability_zone": "ap-northeast-1a",
                "private_dns_name": "ip-10-152-59-198.ap-northeast-1.compute.internal",
                "backends": 5,
                "aws_instance_id": "i-d84cf4da",
                "region": "ap-northeast-1",
                "public_dns_name": "ec2-176-34-13-248.ap-northeast-1.compute.amazonaws.com",
                "layers": [
                  "dump"
                ],
                "id": "e6a12795-52d1-4acc-a047-a8ceef5624a6",
                "hostname": "dump2",
                "ip": "176.34.13.248",
                "private_ip": "10.152.59.198",
                "architecture": "x86_64"
              },
            "stack": {
                "name": "all in one"
              },
            "valid_client_activities": [
              "reboot",
              "stop",
              "setup",
              "configure",
              "update_dependencies",
              "install_dependencies",
              "update_custom_cookbooks",
              "execute_recipes"
            ],
            "ruby_stack": "ruby_enterprise",
            "ruby_version": "1.8.7",
            "deployment": null,
            "rails_stack": {
                "name": null
              },
            "agent_version": "119",
            "applications": [
              {
                  "slug_name": "c5",
                  "application_type": "php",
                  "name": "c5"
                },
              {
                  "slug_name": "wp",
                  "application_type": "php",
                  "name": "wp"
                }
            ],
            "layers": {
                "wp": {
                    "instances": {
                      },
                    "name": "WP",
                    "id": "f32f8e1b-30d1-44a6-8d99-b90dad09958c"
                  },
                "db-master": {
                    "instances": {
                      },
                    "name": "MySQL Master",
                    "id": "e554fc61-25f0-4cde-ab21-0fb890f586b6"
                  },
                "php-app": {
                    "instances": {
                      },
                    "name": "PHP Application Server",
                    "id": "d4ebe207-567f-4547-8e2d-403cd80cb81d"
                  },
                "dump": {
                    "instances": {
                        "dump2": {
                            "instance_type": "m1.small",
                            "availability_zone": "ap-northeast-1a",
                            "private_dns_name": "ip-10-152-59-198.ap-northeast-1.compute.internal",
                            "elastic_ip": null,
                            "status": "online",
                            "created_at": "2013-05-22T14:35:58+00:00",
                            "backends": 5,
                            "aws_instance_id": "i-d84cf4da",
                            "region": "ap-northeast-1",
                            "public_dns_name": "ec2-176-34-13-248.ap-northeast-1.compute.amazonaws.com",
                            "booted_at": "2013-05-22T14:37:12+00:00",
                            "ip": "176.34.13.248",
                            "private_ip": "10.152.59.198"
                          }
                      },
                    "name": "dump",
                    "id": "aab925b1-b2b5-4874-8624-465b538a64b3"
                  },
                "web": {
                    "instances": {
                      },
                    "name": "Web Server",
                    "id": "fd723151-8212-4e81-b033-4374a718596d"
                  }
              },
            "sent_at": 1369234053
          },
        "mysql": {
            "grants_path": "/etc/mysql/grants.sql",
            "socket": "/var/run/mysqld/mysqld.sock",
            "server_root_password": "wiuhfzhstl",
            "conf_dir": "/etc/mysql",
            "pid_file": "/var/run/mysqld/mysqld.pid",
            "confd_dir": "/etc/mysql/conf.d"
          },
        "platform?": [
          "centos",
          "redhat",
          "fedora",
          "amazon"
        ],
        "ssh_users": {
          }
      },
    "name": "dump2.localdomain"
  }
```


## 後記

私もOpsWorksのStack情報を使おうとして、`opsworks-agent-cli`のJSON出力を無理やり`attributes`に起こすOhaiプラグインを書いて**Feedbackから速攻送信**したら『イイネ、でも`node[:opsworks]`に最初からあるんだ』と帰ってきたのを思い出しました。

