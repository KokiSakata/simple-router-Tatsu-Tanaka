#情報ネットワーク学演習II 10/12 レポート課題
===========
学籍番号 33E16011
提出者 田中 達也

## 課題 (ルータの CLI を作ろう)

ルータのコマンドラインインタフェース (CLI) を作ろう。

次の操作ができるコマンドを作ろう。

* ルーティングテーブルの表示
* ルーティングテーブルエントリの追加と削除
* ルータのインタフェース一覧の表示
* そのほか、あると便利な機能

コントローラを操作するコマンドの作りかたは、第3回パッチパネルで作った patch_panel コマンドを参考にしてください。

## 解答
コマンドラインインターフェースを作るために、patch_panelの課題と同様にbinフォルダ内に`コマンド実行用のsimple_router.rb`を作成する．
### ルーティングテーブルの表示
ルーティングテーブルの表示のコマンドを以下のように定義した。
``` 
./bin/simple_router printTable
```
[./bin/simple_router](https://github.com/handai-trema/simple-router-Tatsu-Tanaka/blob/master/bin/simple_router)にルーティングテーブルの表示用のサブコマンドとして以下の記述を行った。
```ruby
  include Pio
  desc 'Prints routing tabel'
  arg_name '@routung_table'
  command :printTable do |c|
    c.desc 'Location to find socket files'
    c.flag [:S, :socket_dir], default_value: Trema::DEFAULT_SOCKET_DIR

    c.action do |_global_options, options, args|
      print "routing table\n"
      print "=========================================\n"
      print "Destination/netmask | Next hop\n"
      print "-----------------------------------------\n"
      @routung_table = Trema.trema_process('SimpleRouter', options[:socket_dir]).controller.return_table()
      @routung_table.each do |each|
        each.each_key do |key|
          print IPv4Address.new(key).to_s, "/", @routung_table.index(each), "\t    | ",each[key].to_s, "\n"
        end
      end
      print "=========================================\n"
    end
  end
```
[./lib/simple_router.rb](https://github.com/handai-trema/simple-router-Tatsu-Tanaka/blob/master/lib/simple_router.rb)内で用意した`return_table()`メソッドを呼び出す。
該当のメソッドは以下である。
```ruby
  def return_table()
    return @routing_table.getDB()
  end
```
ルーティングテーブルはRoutingTableクラス内で保持されている@dbに保存されているため、
[./lib/routing_table.rb](https://github.com/handai-trema/simple-router-Tatsu-Tanaka/blob/master/lib/routing_table.rb)内で用意した`getDB()`メソッドを呼び出す。
該当のメソッドは以下である。
```ruby
  def getDB()
    return @db
  end
```
これにより、`return_table`メソッドの返り値としてルーティングテーブルの内容が返され、それを`@routing_table`に入れている。
ルーティングテーブルの内容を表示するために、宛先アドレス/ネットマスク長、転送先の順に表示する。
RoutingTableクラス内の@dbで、宛先アドレスはハッシュのキーとして保存されており、表示する際に文字列として表示する。
ネットマスク長は配列のインデックスとして保存されているので、indexメソッドを用いて取得する。
転送先アドレスも同様に文字列として表示する。

#### 実行結果
起動時のルーティングテーブルは`simple_router.conf`で定義されている。
```
  ROUTES = [
    {
      destination: '0.0.0.0',
      netmask_length: 0,
      next_hop: '192.168.1.2'
    }
  ]
```
実行結果を以下に示す。
```
ensyuu2@ensyuu2-VirtualBox:~/simple-router-Tatsu-Tanaka$ ./bin/simple_router printTable
routing table
=========================================
Destination/netmask | Next hop
-----------------------------------------
0.0.0.0/0	    | 192.168.1.2
=========================================
```
以降の課題のおいて、このコマンドを用いて、正しく実行できていることを確認する。

### ルーティングテーブルエントリの追加と削除
ルーティングテーブルエントリの追加と削除のコマンドを以下のように定義した。
``` 
./bin/simple_router addTable [宛先アドレス] [ネットマスク長] [転送先アドレス]
```
``` 
./bin/simple_router deleteTable [宛先アドレス] [ネットマスク長]
```
[./bin/simple_router](https://github.com/handai-trema/simple-router-Tatsu-Tanaka/blob/master/bin/simple_router)にルーティングテーブルの表示用のサブコマンドとして以下の記述を行った。
```ruby
  desc 'add routing tabel'
  arg_name 'destination netmask nexthop'
  command :addTable do |c|
    c.desc 'Location to find socket files'
    c.flag [:S, :socket_dir], default_value: Trema::DEFAULT_SOCKET_DIR

    c.action do |_global_options, options, args|
      destination = args[0]
      netmask = args[1]
      nexthop = args[2]
      Trema.trema_process('SimpleRouter', options[:socket_dir]).controller.add_routing_tabel(destination, netmask, nexthop)
    end
  end

  desc 'delete routing tabel'
  arg_name 'destination netmask'
  command :deleteTable do |c|
    c.desc 'Location to find socket files'
    c.flag [:S, :socket_dir], default_value: Trema::DEFAULT_SOCKET_DIR

    c.action do |_global_options, options, args|
      destination = args[0]
      netmask = args[1]
      Trema.trema_process('SimpleRouter', options[:socket_dir]).controller.delete_routing_tabel(destination, netmask)
    end
  end
```
[./lib/simple_router.rb](https://github.com/handai-trema/simple-router-Tatsu-Tanaka/blob/master/lib/simple_router.rb)内で用意した追加用の`add_routing_tabel`メソッド、削除用の`delete_routing_table`メソッドを呼び出す。
該当のメソッドは以下である。
```ruby
  def add_routing_tabel(destination, netmask, nexthop)
    options = {:destination => destination, :netmask_length => netmask.to_i, :next_hop => nexthop}
    @routing_table.add(options)
  end

  def delete_routing_tabel(destination, netmask)
    options = {:destination => destination, :netmask_length => netmask.to_i}
    @routing_table.delete(options)
  end
```
RoutingTableクラス内ですでに用意されているaddメソッドに加え、deleteメソッドを作成した。
[./lib/routing_table.rb](https://github.com/handai-trema/simple-router-Tatsu-Tanaka/blob/master/lib/routing_table.rb)内の該当部分を以下に示す。
```ruby
  def add(options)
    netmask_length = options.fetch(:netmask_length)
    prefix = IPv4Address.new(options.fetch(:destination)).mask(netmask_length)
    @db[netmask_length][prefix.to_i] = IPv4Address.new(options.fetch(:next_hop))
  end

  def delete(options)
    netmask_length = options.fetch(:netmask_length)
    prefix = IPv4Address.new(options.fetch(:destination)).mask(netmask_length)
    @db[netmask_length].delete(prefix.to_i)
  end
```
#### 実行結果
実行結果を以下の手順で示す。
* ルーティングテーブルの追加
* ルーティングテーブルの表示
* ルーティングテーブルの削除
* ルーティングテーブルの表示
```
ensyuu2@ensyuu2-VirtualBox:~/simple-router-Tatsu-Tanaka$ ./bin/simple_router addTable 192.168.1.0 24 192.168.1.1
ensyuu2@ensyuu2-VirtualBox:~/simple-router-Tatsu-Tanaka$ ./bin/simple_router printTable
routing table
=========================================
Destination/netmask | Next hop
-----------------------------------------
0.0.0.0/0	    | 192.168.1.2
192.168.1.0/24	    | 192.168.1.1
=========================================
ensyuu2@ensyuu2-VirtualBox:~/simple-router-Tatsu-Tanaka$ ./bin/simple_router deleteTable 192.168.1.0 24
ensyuu2@ensyuu2-VirtualBox:~/simple-router-Tatsu-Tanaka$ ./bin/simple_router printTable
routing table
=========================================
Destination/netmask | Next hop
-----------------------------------------
0.0.0.0/0	    | 192.168.1.2
=========================================
```
正しく実装できていることを確認した。


### ルータのインタフェース一覧の表示
ルーティングテーブルエントリの追加と削除のコマンドを以下のように定義した。
``` 
./bin/simple_router printInterface
```
[./bin/simple_router](https://github.com/handai-trema/simple-router-Tatsu-Tanaka/blob/master/bin/simple_router)にルーティングテーブルの表示用のサブコマンドとして以下の記述を行った。
```ruby
  desc 'Prints interface'
  arg_name 'interface'
  command :printInterface do |c|
    c.desc 'Location to find socket files'
    c.flag [:S, :socket_dir], default_value: Trema::DEFAULT_SOCKET_DIR

    c.action do |_global_options, options, args|
      print "interface list\n"
      print "=============================================\n"
      print "port | mac address  \t | ip address/netmask\n"
      print "---------------------------------------------\n"
      interface = Trema.trema_process('SimpleRouter', options[:socket_dir]).controller.return_interface()
      interface.each do |each|
        print each[:port_number].to_s,"    | ", each[:mac_address], " | ", each[:ip_address]+"/"+each[:netmask_length].to_s,"\n"
      end
      print "=============================================\n"
    end
  end
```
[./lib/simple_router.rb](https://github.com/handai-trema/simple-router-Tatsu-Tanaka/blob/master/lib/simple_router.rb)内で用意した追加用の`return_interface`メソッドを呼び出す。
該当のメソッドは以下である。
```ruby
  def return_interface()
    interfaceList = Array.new()
    Interface.all.each do |each|
      interfaceList << {:port_number => each.port_number, :mac_address => each.mac_address.to_s, :ip_address => each.ip_address.value.to_s, :netmask_length => each.netmask_length}
    end
    return interfaceList
  end
```
用意されているlib/interface.rbでインターフェースについて定義されているのでそれを参考にした。

#### 実行結果
起動時のインターフェースは`simple_router.conf`で定義されている。
```
  INTERFACES = [
    {
      port: 1,
      mac_address: '01:01:01:01:01:01',
      ip_address: '192.168.1.1',
      netmask_length: 24
    },
    {
      port: 2,
      mac_address: '02:02:02:02:02:02',
      ip_address: '192.168.2.1',
      netmask_length: 24
    }
  ]
```
実行結果を以下に示す。
```
ensyuu2@ensyuu2-VirtualBox:~/simple-router-Tatsu-Tanaka$ ./bin/simple_router printInterface
interface list
=============================================
port | mac address  	 | ip address/netmask
---------------------------------------------
1    | 01:01:01:01:01:01 | 192.168.1.1/24
2    | 02:02:02:02:02:02 | 192.168.2.1/24
=============================================
```
正しく実装できていることを確認できた。

##参考文献
- デビッド・トーマス+アンドリュー・ハント(2001)「プログラミング Ruby」ピアソン・エデュケーション.  
- [テキスト: 13章 "ルータ (後編)"](http://yasuhito.github.io/trema-book/#router_part2)  
