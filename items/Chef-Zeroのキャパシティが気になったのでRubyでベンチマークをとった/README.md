<!-- too_old -->
> **この記事は最終更新から1年以上経過しています。** 気をつけてね。


実行したコードはこちら。 [higanworks/chefzero-benchmark](https://github.com/higanworks/chefzero-benchmark "higanworks/chefzero-benchmark")


[The Ruby Toolbox - Benchmarking](https://www.ruby-toolbox.com/categories/Benchmarking "The Ruby Toolbox - Benchmarking")からなにかしら結果が見やすいとか使いやすいのはないかと見てみたけど、結局普通の`Benchmark.measure`を使い、次のやり方で計測しました。

1. Chef-Zefoを起動する
1. ダミーのノードを[Fauxhai](https://github.com/customink/fauxhai)で生成する
1. [Ridley](https://github.com/RiotGames/ridley)でノードをChef-Zeroに登録する
    - 登録数は [50, 100, 500, 1000, 5000] の5パターン
1. 次の3つを計測
    - ノードを1つ抽出する
    - サーチで単純なクエリにマッチするNodeを抽出する
    - サーチでワイルドカードを含むクエリにマッチするNodeを抽出する

Chef-Zeroの保存方式(データストア)は標準のオンメモリと、実際のディレクトリを直接使用するファイルシステム(※こっちに期待)の2種類。

ベンチマークはこんな条件の下で行った。

```
MacBook Pro Retina, 13-inch, Late 2013
Processor: 2.8 GHz Intel Core i7
Memory: 16 GB 1600 MHz DDR3

RUBY_VERSION #=> 2.1.2
RbConfig::CONFIG["PATCHLEVEL"] #=> 95
ChefZero::VERSION #=> 3.0
```


## 結果のまとめ

どちらのデータストアでも、単体を取得する分には特に劣化がなかった。
オンメモリなら1000ノードのサーチも2秒程度、流石に5000ノードあったら10秒以上かかっていたけど許容範囲。

本命のファイルシステムだと、500ノードを超えたあたりからサーチが徐々に厳しくなっていく。
5000ノードでサーチすると流石にRidley標準のタイムアウトで終了した。

でもサーチを使わなければ、動かすマシンにもよるが5000程度のオブジェクトがあってもへっちゃら感です。


## ベンチマークのソース

`Benchmark#measure`を使う部分はまとめるべきだったと思いつつベタ書き。


```ruby:Rakefile
require 'chef_zero/server'
require 'ridley'
require 'fauxhai'
require 'benchmark'
require 'tempfile'
require 'faker'

TIMES = [50, 100, 500, 1000, 5000]

def ridley_init
  f = Tempfile.new('pem')
  File.write(f.path, OpenSSL::PKey::RSA.new(2048))

  ridley = Ridley.new(
    server_url: "http://localhost:4000",
    client_name: "reset",
    client_key: f.path
  )
  ridley.logger.level = 3
  ridley
end

def create_fake_node(ridley, count)
  count.times do |t|
    obj = ridley.node.new
    obj.name = [Faker::Internet.domain_word + t.to_s , Faker::Internet.domain_name].join('.')
    @nodes << obj.name
    obj.chef_environment = get_env_sample
    obj.automatic = Fauxhai.mock(platform: 'ubuntu', version: '12.04').data
    ridley.node.create(obj)
  end
end

def get_env_sample
  [
    'development',
    'production',
    'office',
    'resort'
  ].sample
end

def version
  puts '============================'
  puts ['RUBY_VERSION', RUBY_VERSION].join(' #=> ')
  puts ['RbConfig::CONFIG["PATCHLEVEL"]', RbConfig::CONFIG['PATCHLEVEL']].join(' #=> ')
  puts ['ChefZero::VERSION', ChefZero::VERSION].join(' #=> ')
  puts '============================'
end

desc 'run benchmark both mem_store and disk_store.'
task :default do
  Rake::Task['bench:mem'].invoke
  Rake::Task['bench:disk'].invoke
end


desc 'memory_store benchmack'
namespace :bench do
  task :mem do
    version
    server = ChefZero::Server.new(port: 4000)
    server.start_background

    ridley = ridley_init

    TIMES.each do |time|
      puts "Preparing fake node objects upto #{time}..."
      ## Warmup
      @nodes = []
      puts Benchmark::CAPTION
      puts Benchmark.measure {
        create_fake_node(ridley, time)
      }
      puts "Done!"
      puts "============================"

      puts "Start benchmark for pick up one node 5 times of #{time} nodes (on memory)"
      5.times do
        puts Benchmark.measure {
          arr =  ridley.node.find(@nodes.sample)
          puts Benchmark::CAPTION
        }
      end

      puts "Start benchmark for search 5 times by simple match query of #{time} nodes (on memory)"
      5.times do
        puts Benchmark.measure {
          arr =  ridley.search(:node, "chef_environment:#{get_env_sample}")
          puts "--------Matched #{arr.size} nodes of #{time} objects"
          puts Benchmark::CAPTION
        }
      end
      puts "Start benchmark for search 5 times with wildcard match query of #{time} nodes (on memory)"
      5.times do
        puts Benchmark.measure {
          arr =  ridley.search(:node, "name:*#{Faker::Internet.domain_suffix}")
          puts "--------Matched #{arr.size} nodes of #{time} objects"
          puts Benchmark::CAPTION
        }
      end
      puts "Done! Data will be cleared."
      puts "============================\n\n"
      server.clear_data
    end
    server.stop
  end

  desc 'disk_store benchmack'
  task :disk do
    require 'chef/chef_fs/chef_fs_data_store'
    require 'chef/chef_fs/config'
    require 'chef_zero/data_store/raw_file_store'

    version

    Dir.mktmpdir do |tmpdir|
      puts "Temporary data_store was created to #{tmpdir}."
      Chef::Config[:chef_repo_path] = tmpdir
      chef_fs = Chef::ChefFS::Config.new.local_fs
      chef_fs.write_pretty_json = true
      data_store = Chef::ChefFS::ChefFSDataStore.new(chef_fs)

      server_options = {}
      server_options[:data_store] = data_store
      server_options[:port] = 4000

      server = ChefZero::Server.new(server_options)
      server.start_background

      ridley = ridley_init

      TIMES.each do |time|
        puts "Preparing fake node objects upto #{time}..."
        ## Warmup
        @nodes = []
        puts Benchmark::CAPTION
        puts Benchmark.measure {
          create_fake_node(ridley, time)
        }
        puts "Done!"
        puts "============================"

        puts "Start benchmark for pick up one node 5 times of #{time} nodes (on filesystem)"
        5.times do
          puts Benchmark.measure {
            arr =  ridley.node.find(@nodes.sample)
            puts Benchmark::CAPTION
          }
        end

        puts "Start benchmark for search 5 times by simple match query of #{time} nodes (on filesystem)"
        5.times do
          puts Benchmark.measure {
            arr =  ridley.search(:node, "chef_environment:#{get_env_sample}")
            puts "--------Matched #{arr.size} nodes of #{time} objects"
            puts Benchmark::CAPTION
          }
        end
        puts "Start benchmark for search 5 times with wildcard match query of #{time} nodes (on filesystem)"
        5.times do
          puts Benchmark.measure {
            arr =  ridley.search(:node, "name:*#{Faker::Internet.domain_suffix}")
            puts "--------Matched #{arr.size} nodes of #{time} objects"
            puts Benchmark::CAPTION
          }
        end

        puts "Done! Data will be cleared."
        puts "============================\n\n"

        chef_fs.child_paths.each_value do |dirs|
          dirs.each do |dir|
            if File.exists?(dir)
              FileUtils.rm(Dir.glob("#{dir}/*.json"))
            end
          end
        end
        server.clear_data
      end
      server.stop
    end
  end
end
```


## ベンチマークの実行結果



```shell:
$ rake 
============================
RUBY_VERSION #=> 2.1.2
RbConfig::CONFIG["PATCHLEVEL"] #=> 95
ChefZero::VERSION #=> 3.0
============================
Preparing fake node objects upto 50...
      user     system      total        real
  0.750000   0.070000   0.820000 (  1.190110)
Done!
============================
Start benchmark for pick up one node 5 times of 50 nodes (on memory)
      user     system      total        real
  0.010000   0.000000   0.010000 (  0.009239)
      user     system      total        real
  0.010000   0.000000   0.010000 (  0.009327)
      user     system      total        real
  0.010000   0.000000   0.010000 (  0.009776)
      user     system      total        real
  0.010000   0.000000   0.010000 (  0.009797)
      user     system      total        real
  0.000000   0.000000   0.000000 (  0.009014)
Start benchmark for search 5 times by simple match query of 50 nodes (on memory)
--------Matched 13 nodes of 50 objects
      user     system      total        real
  0.150000   0.010000   0.160000 (  0.153010)
--------Matched 16 nodes of 50 objects
      user     system      total        real
  0.130000   0.010000   0.140000 (  0.133699)
--------Matched 11 nodes of 50 objects
      user     system      total        real
  0.120000   0.010000   0.130000 (  0.132803)
--------Matched 11 nodes of 50 objects
      user     system      total        real
  0.110000   0.000000   0.110000 (  0.114197)
--------Matched 10 nodes of 50 objects
      user     system      total        real
  0.150000   0.010000   0.160000 (  0.156211)
Start benchmark for search 5 times with wildcard match query of 50 nodes (on memory)
--------Matched 10 nodes of 50 objects
      user     system      total        real
  0.110000   0.000000   0.110000 (  0.112973)
--------Matched 9 nodes of 50 objects
      user     system      total        real
  0.110000   0.010000   0.120000 (  0.117174)
--------Matched 10 nodes of 50 objects
      user     system      total        real
  0.110000   0.000000   0.110000 (  0.120937)
--------Matched 6 nodes of 50 objects
      user     system      total        real
  0.120000   0.000000   0.120000 (  0.117076)
--------Matched 6 nodes of 50 objects
      user     system      total        real
  0.100000   0.010000   0.110000 (  0.110966)
Done! Data will be cleared.
============================

Preparing fake node objects upto 100...
      user     system      total        real
  1.170000   0.130000   1.300000 (  1.319404)
Done!
============================
Start benchmark for pick up one node 5 times of 100 nodes (on memory)
      user     system      total        real
  0.000000   0.010000   0.010000 (  0.007963)
      user     system      total        real
  0.010000   0.000000   0.010000 (  0.008128)
      user     system      total        real
  0.010000   0.000000   0.010000 (  0.008020)
      user     system      total        real
  0.000000   0.000000   0.000000 (  0.007828)
      user     system      total        real
  0.010000   0.000000   0.010000 (  0.008201)
Start benchmark for search 5 times by simple match query of 100 nodes (on memory)
--------Matched 16 nodes of 100 objects
      user     system      total        real
  0.250000   0.010000   0.260000 (  0.258547)
--------Matched 22 nodes of 100 objects
      user     system      total        real
  0.210000   0.000000   0.210000 (  0.215202)
--------Matched 22 nodes of 100 objects
      user     system      total        real
  0.250000   0.010000   0.260000 (  0.264075)
--------Matched 22 nodes of 100 objects
      user     system      total        real
  0.230000   0.000000   0.230000 (  0.239582)
--------Matched 34 nodes of 100 objects
      user     system      total        real
  0.260000   0.010000   0.270000 (  0.268535)
Start benchmark for search 5 times with wildcard match query of 100 nodes (on memory)
--------Matched 12 nodes of 100 objects
      user     system      total        real
  0.230000   0.020000   0.250000 (  0.240597)
--------Matched 22 nodes of 100 objects
      user     system      total        real
  0.320000   0.020000   0.340000 (  0.341606)
--------Matched 14 nodes of 100 objects
      user     system      total        real
  0.240000   0.000000   0.240000 (  0.243652)
--------Matched 12 nodes of 100 objects
      user     system      total        real
  0.190000   0.010000   0.200000 (  0.197043)
--------Matched 19 nodes of 100 objects
      user     system      total        real
  0.210000   0.000000   0.210000 (  0.215121)
Done! Data will be cleared.
============================

Preparing fake node objects upto 500...
      user     system      total        real
  5.730000   0.660000   6.390000 (  6.489995)
Done!
============================
Start benchmark for pick up one node 5 times of 500 nodes (on memory)
      user     system      total        real
  0.010000   0.000000   0.010000 (  0.008062)
      user     system      total        real
  0.000000   0.000000   0.000000 (  0.007949)
      user     system      total        real
  0.010000   0.000000   0.010000 (  0.007695)
      user     system      total        real
  0.010000   0.000000   0.010000 (  0.007941)
      user     system      total        real
  0.010000   0.010000   0.020000 (  0.008567)
Start benchmark for search 5 times by simple match query of 500 nodes (on memory)
--------Matched 141 nodes of 500 objects
      user     system      total        real
  1.190000   0.020000   1.210000 (  1.224797)
--------Matched 134 nodes of 500 objects
      user     system      total        real
  1.260000   0.060000   1.320000 (  1.316611)
--------Matched 141 nodes of 500 objects
      user     system      total        real
  1.300000   0.030000   1.330000 (  1.329287)
--------Matched 134 nodes of 500 objects
      user     system      total        real
  1.300000   0.030000   1.330000 (  1.333752)
--------Matched 112 nodes of 500 objects
      user     system      total        real
  1.330000   0.040000   1.370000 (  1.369286)
Start benchmark for search 5 times with wildcard match query of 500 nodes (on memory)
--------Matched 88 nodes of 500 objects
      user     system      total        real
  1.040000   0.020000   1.060000 (  1.059881)
--------Matched 86 nodes of 500 objects
      user     system      total        real
  1.200000   0.050000   1.250000 (  1.254659)
--------Matched 88 nodes of 500 objects
      user     system      total        real
  1.170000   0.020000   1.190000 (  1.189626)
--------Matched 86 nodes of 500 objects
      user     system      total        real
  1.080000   0.020000   1.100000 (  1.099927)
--------Matched 76 nodes of 500 objects
      user     system      total        real
  1.240000   0.030000   1.270000 (  1.264090)
Done! Data will be cleared.
============================

Preparing fake node objects upto 1000...
      user     system      total        real
 11.460000   1.310000  12.770000 ( 12.823895)
Done!
============================
Start benchmark for pick up one node 5 times of 1000 nodes (on memory)
      user     system      total        real
  0.010000   0.000000   0.010000 (  0.007845)
      user     system      total        real
  0.000000   0.000000   0.000000 (  0.008117)
      user     system      total        real
  0.010000   0.000000   0.010000 (  0.007736)
      user     system      total        real
  0.010000   0.000000   0.010000 (  0.007951)
      user     system      total        real
  0.010000   0.000000   0.010000 (  0.009608)
Start benchmark for search 5 times by simple match query of 1000 nodes (on memory)
--------Matched 238 nodes of 1000 objects
      user     system      total        real
  2.460000   0.040000   2.500000 (  2.514587)
--------Matched 247 nodes of 1000 objects
      user     system      total        real
  2.540000   0.050000   2.590000 (  2.595222)
--------Matched 261 nodes of 1000 objects
      user     system      total        real
  2.750000   0.060000   2.810000 (  2.810951)
--------Matched 247 nodes of 1000 objects
      user     system      total        real
  2.550000   0.050000   2.600000 (  2.621313)
--------Matched 238 nodes of 1000 objects
      user     system      total        real
  2.650000   0.050000   2.700000 (  2.719356)
Start benchmark for search 5 times with wildcard match query of 1000 nodes (on memory)
--------Matched 174 nodes of 1000 objects
      user     system      total        real
  2.540000   0.050000   2.590000 (  2.600654)
--------Matched 164 nodes of 1000 objects
      user     system      total        real
  2.850000   0.080000   2.930000 (  2.945433)
--------Matched 174 nodes of 1000 objects
      user     system      total        real
  3.050000   0.060000   3.110000 (  3.134419)
--------Matched 185 nodes of 1000 objects
      user     system      total        real
  2.520000   0.050000   2.570000 (  2.582858)
--------Matched 174 nodes of 1000 objects
      user     system      total        real
  2.410000   0.040000   2.450000 (  2.453993)
Done! Data will be cleared.
============================

Preparing fake node objects upto 5000...
      user     system      total        real
 58.740000   6.480000  65.220000 ( 65.438612)
Done!
============================
Start benchmark for pick up one node 5 times of 5000 nodes (on memory)
      user     system      total        real
  0.010000   0.000000   0.010000 (  0.008116)
      user     system      total        real
  0.000000   0.000000   0.000000 (  0.008034)
      user     system      total        real
  0.010000   0.000000   0.010000 (  0.008345)
      user     system      total        real
  0.010000   0.000000   0.010000 (  0.008565)
      user     system      total        real
  0.010000   0.000000   0.010000 (  0.008128)
Start benchmark for search 5 times by simple match query of 5000 nodes (on memory)
--------Matched 1234 nodes of 5000 objects
      user     system      total        real
 21.720000   0.440000  22.160000 ( 22.189359)
--------Matched 1234 nodes of 5000 objects
      user     system      total        real
 16.100000   0.330000  16.430000 ( 16.437395)
--------Matched 1234 nodes of 5000 objects
      user     system      total        real
 12.600000   0.350000  12.950000 ( 12.960188)
--------Matched 1280 nodes of 5000 objects
      user     system      total        real
 13.010000   0.320000  13.330000 ( 13.400405)
--------Matched 1280 nodes of 5000 objects
      user     system      total        real
 13.110000   0.300000  13.410000 ( 13.486141)
Start benchmark for search 5 times with wildcard match query of 5000 nodes (on memory)
--------Matched 852 nodes of 5000 objects
      user     system      total        real
 13.220000   0.460000  13.680000 ( 13.835438)
--------Matched 862 nodes of 5000 objects
      user     system      total        real
 12.290000   0.300000  12.590000 ( 12.635503)
--------Matched 871 nodes of 5000 objects
      user     system      total        real
 13.030000   0.250000  13.280000 ( 13.518441)
--------Matched 862 nodes of 5000 objects
      user     system      total        real
 12.390000   0.240000  12.630000 ( 12.693962)
--------Matched 813 nodes of 5000 objects
      user     system      total        real
 11.980000   0.220000  12.200000 ( 12.211569)
Done! Data will be cleared.
============================

============================
RUBY_VERSION #=> 2.1.2
RbConfig::CONFIG["PATCHLEVEL"] #=> 95
ChefZero::VERSION #=> 3.0
============================
Temporary data_store was created to /var/folders/s9/kwcs_8ln0d32v_n4hdfb7td00000gn/T/d20140826-31513-1innb9o.
Preparing fake node objects upto 50...
      user     system      total        real
  0.690000   0.090000   0.780000 (  0.795218)
Done!
============================
Start benchmark for pick up one node 5 times of 50 nodes (on filesystem)
      user     system      total        real
  0.010000   0.000000   0.010000 (  0.007690)
      user     system      total        real
  0.000000   0.000000   0.000000 (  0.007726)
      user     system      total        real
  0.010000   0.000000   0.010000 (  0.007594)
      user     system      total        real
  0.010000   0.000000   0.010000 (  0.007664)
      user     system      total        real
  0.000000   0.000000   0.000000 (  0.009790)
Start benchmark for search 5 times by simple match query of 50 nodes (on filesystem)
--------Matched 0 nodes of 50 objects
      user     system      total        real
  0.050000   0.010000   0.060000 (  0.055999)
--------Matched 0 nodes of 50 objects
      user     system      total        real
  0.110000   0.020000   0.130000 (  0.119940)
--------Matched 0 nodes of 50 objects
      user     system      total        real
  0.050000   0.010000   0.060000 (  0.056107)
--------Matched 0 nodes of 50 objects
      user     system      total        real
  0.050000   0.010000   0.060000 (  0.067170)
--------Matched 0 nodes of 50 objects
      user     system      total        real
  0.060000   0.010000   0.070000 (  0.073780)
Start benchmark for search 5 times with wildcard match query of 50 nodes (on filesystem)
--------Matched 8 nodes of 50 objects
      user     system      total        real
  0.040000   0.010000   0.050000 (  0.052666)
--------Matched 7 nodes of 50 objects
      user     system      total        real
  0.050000   0.010000   0.060000 (  0.055499)
--------Matched 7 nodes of 50 objects
      user     system      total        real
  0.040000   0.010000   0.050000 (  0.052032)
--------Matched 4 nodes of 50 objects
      user     system      total        real
  0.160000   0.010000   0.170000 (  0.165832)
--------Matched 6 nodes of 50 objects
      user     system      total        real
  0.040000   0.010000   0.050000 (  0.052942)
Done! Data will be cleared.
============================

Preparing fake node objects upto 100...
      user     system      total        real
  1.320000   0.160000   1.480000 (  1.497286)
Done!
============================
Start benchmark for pick up one node 5 times of 100 nodes (on filesystem)
      user     system      total        real
  0.010000   0.010000   0.020000 (  0.008849)
      user     system      total        real
  0.010000   0.000000   0.010000 (  0.008563)
      user     system      total        real
  0.000000   0.000000   0.000000 (  0.008509)
      user     system      total        real
  0.010000   0.000000   0.010000 (  0.008801)
      user     system      total        real
  0.010000   0.000000   0.010000 (  0.011175)
Start benchmark for search 5 times by simple match query of 100 nodes (on filesystem)
--------Matched 0 nodes of 100 objects
      user     system      total        real
  0.220000   0.030000   0.250000 (  0.246197)
--------Matched 0 nodes of 100 objects
      user     system      total        real
  0.130000   0.030000   0.160000 (  0.163799)
--------Matched 0 nodes of 100 objects
      user     system      total        real
  0.260000   0.030000   0.290000 (  0.284852)
--------Matched 0 nodes of 100 objects
      user     system      total        real
  0.140000   0.030000   0.170000 (  0.171276)
--------Matched 0 nodes of 100 objects
      user     system      total        real
  0.250000   0.030000   0.280000 (  0.276693)
Start benchmark for search 5 times with wildcard match query of 100 nodes (on filesystem)
--------Matched 15 nodes of 100 objects
      user     system      total        real
  0.150000   0.030000   0.180000 (  0.188616)
--------Matched 19 nodes of 100 objects
      user     system      total        real
  0.260000   0.030000   0.290000 (  0.291682)
--------Matched 19 nodes of 100 objects
      user     system      total        real
  0.150000   0.040000   0.190000 (  0.189395)
--------Matched 19 nodes of 100 objects
      user     system      total        real
  0.330000   0.040000   0.370000 (  0.380891)
--------Matched 21 nodes of 100 objects
      user     system      total        real
  0.150000   0.030000   0.180000 (  0.173191)
Done! Data will be cleared.
============================

Preparing fake node objects upto 500...
      user     system      total        real
  6.940000   0.900000   7.840000 (  7.878502)
Done!
============================
Start benchmark for pick up one node 5 times of 500 nodes (on filesystem)
      user     system      total        real
  0.020000   0.000000   0.020000 (  0.014835)
      user     system      total        real
  0.010000   0.000000   0.010000 (  0.015202)
      user     system      total        real
  0.080000   0.010000   0.090000 (  0.086860)
      user     system      total        real
  0.010000   0.000000   0.010000 (  0.015377)
      user     system      total        real
  0.020000   0.000000   0.020000 (  0.015287)
Start benchmark for search 5 times by simple match query of 500 nodes (on filesystem)
--------Matched 0 nodes of 500 objects
      user     system      total        real
  4.500000   0.810000   5.310000 (  5.316760)
--------Matched 0 nodes of 500 objects
      user     system      total        real
  4.530000   0.790000   5.320000 (  5.314164)
--------Matched 0 nodes of 500 objects
      user     system      total        real
  4.470000   0.780000   5.250000 (  5.270624)
--------Matched 0 nodes of 500 objects
      user     system      total        real
  4.440000   0.780000   5.220000 (  5.215047)
--------Matched 0 nodes of 500 objects
      user     system      total        real
  4.830000   0.830000   5.660000 (  5.705454)
Start benchmark for search 5 times with wildcard match query of 500 nodes (on filesystem)
--------Matched 101 nodes of 500 objects
      user     system      total        real
  4.410000   0.760000   5.170000 (  5.172413)
--------Matched 75 nodes of 500 objects
      user     system      total        real
  4.540000   0.760000   5.300000 (  5.332210)
--------Matched 74 nodes of 500 objects
      user     system      total        real
  4.500000   0.760000   5.260000 (  5.259890)
--------Matched 89 nodes of 500 objects
      user     system      total        real
  4.430000   0.760000   5.190000 (  5.200891)
--------Matched 101 nodes of 500 objects
      user     system      total        real
  4.520000   0.790000   5.310000 (  5.326836)
Done! Data will be cleared.
============================

Preparing fake node objects upto 1000...
      user     system      total        real
 14.400000   1.850000  16.250000 ( 16.261022)
Done!
============================
Start benchmark for pick up one node 5 times of 1000 nodes (on filesystem)
      user     system      total        real
  0.020000   0.000000   0.020000 (  0.022240)
      user     system      total        real
  0.020000   0.010000   0.030000 (  0.023544)
      user     system      total        real
  0.020000   0.000000   0.020000 (  0.025304)
      user     system      total        real
  0.020000   0.010000   0.030000 (  0.025084)
      user     system      total        real
  0.020000   0.000000   0.020000 (  0.027034)
Start benchmark for search 5 times by simple match query of 1000 nodes (on filesystem)
--------Matched 0 nodes of 1000 objects
      user     system      total        real
 18.240000   3.240000  21.480000 ( 21.577591)
--------Matched 0 nodes of 1000 objects
      user     system      total        real
 17.410000   3.020000  20.430000 ( 20.477257)
--------Matched 0 nodes of 1000 objects
      user     system      total        real
 17.430000   3.100000  20.530000 ( 20.582965)
--------Matched 0 nodes of 1000 objects
      user     system      total        real
 17.150000   3.000000  20.150000 ( 20.186336)
--------Matched 0 nodes of 1000 objects
      user     system      total        real
 17.700000   3.100000  20.800000 ( 20.857837)
Start benchmark for search 5 times with wildcard match query of 1000 nodes (on filesystem)
--------Matched 171 nodes of 1000 objects
      user     system      total        real
 17.660000   3.070000  20.730000 ( 20.761662)
--------Matched 171 nodes of 1000 objects
      user     system      total        real
 17.200000   3.030000  20.230000 ( 20.273779)
--------Matched 174 nodes of 1000 objects
      user     system      total        real
 17.210000   3.000000  20.210000 ( 20.221381)
--------Matched 159 nodes of 1000 objects
      user     system      total        real
 17.470000   3.030000  20.500000 ( 20.542159)
--------Matched 171 nodes of 1000 objects
      user     system      total        real
 17.400000   3.020000  20.420000 ( 20.460623)
Done! Data will be cleared.
============================

Preparing fake node objects upto 5000...
      user     system      total        real
 69.430000   9.860000  79.290000 ( 79.626666)
Done!
============================
Start benchmark for pick up one node 5 times of 5000 nodes (on filesystem)
      user     system      total        real
  0.080000   0.010000   0.090000 (  0.093334)
      user     system      total        real
  0.070000   0.020000   0.090000 (  0.091920)
      user     system      total        real
  0.080000   0.020000   0.100000 (  0.090885)
      user     system      total        real
  0.180000   0.010000   0.190000 (  0.196649)
      user     system      total        real
  0.070000   0.020000   0.090000 (  0.097520)
Start benchmark for search 5 times by simple match query of 5000 nodes (on filesystem)
E, [2014-08-26T20:23:35.452344 #31513] ERROR -- : Actor crashed!
Ridley::Errors::TimeoutError: too many connection resets (due to Net::ReadTimeout - Net::ReadTimeout) after 0 requests on 2597410980, last used 125.772078 seconds ago
```
