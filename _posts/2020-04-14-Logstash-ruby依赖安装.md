# Logstash ruby依赖安装

## 1、背景

  在编写logstash filter中，可以使用ruby语言来进行脚本的扩展，但是在使用中，往往会用到基本库以外的依赖。在普通的ruby环境下，直接使用 `gem install 'requirement'`即可完成依赖的安装。但是于logstash，它并非直接调用系统的ruby环境，而是自带了一个ruby环境，并且官方描述中表面，他们使用ruby官方Gem库，我们可以借此来安装自己的依赖。此处顺便记录自己开发的一个简单的Maxmind ruby filter作为距离

## 2、过程记录

### 2.1 ruby代码编写

```ruby
require 'maxmind/db'

asn_reader = MaxMind::DB.new('asset/GeoLite2-ASN.mmdb',mode: MaxMind::DB::MODE_MEMORY)
city_reader = MaxMind::DB.new('asset/GeoLite2-City.mmdb',mode: MaxMind::DB::MODE_MEMORY)

asn_record = asn_reader.get("1.1.1.1")
city_record = city_reader.get("1.1.1.1")

puts city_record
puts asn_record
```

此处贴上测试代码，这是一串非常简单的ruby测试代码。测试完毕通过后，我们按照官方提供的模板，将其改写为logstash接收的script：

```ruby
def register(params)
  require 'maxmind/db'
  @asn_reader = MaxMind::DB.new('asset/maxmind/GeoLite2-ASN.mmdb',mode: MaxMind::DB::MODE_MEMORY)
  @city_reader = MaxMind::DB.new('asset/GeoLite2-City.mmdb',mode: MaxMind::DB::MODE_MEMORY)
end

def filter(event)
  clientip = event.get("srcip")
  asn_record = @asn_reader.get(clientip)
  city_record = @city_reader.get(clientip)

  event.set("asn", asn_record)
  event.set("geoip", city_record)

  return [event]
end
```

此处需要注意几个细节

1.  在register代码块中，进行maxmind-db的读取，这样可以避免对于文件的重复读取
2.  @变量名 可以将变量传递至filter中进行使用
3.  return [] 返回一个值非常重要，并且需要是list的形式。不然最终无法正确返回参数。

### 2.2 error出现

`[main] Logstash - java.lang.IllegalStateException: Logstash stopped processing because of an error: (GemNotFound) Could not find maxmind-db-1.1.0 in any of the sources`

Here comes a new challenger!

原因很简单，找不到依赖包，

### 2.3 问题解决

`/usr/share/logstash/bin/ruby -S gem install maxmind-db`

 通过这种方式即可安装相应的依赖了，于此同时需要去/usr/share/logstash/Gemfile中新增一行 

`gem 'maxmind-db'`

