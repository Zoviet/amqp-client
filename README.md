# AMQP Client #

It is a fork of :

    https://github.com/mengz0/amqp
    https://github.com/ZigzagAK/amqp
    https://github.com/4mig4/lua-amqp
    https://github.com/gsdenys/amqp-client

This library can be used with LuaJIT and does not have to be used only in OpenResty.

This library use lua socket and ***don't use cqueues***.

***Added Reject, requeue, multiple, queue arguments and remove cqueues connects:***

If requeue option sets in false then acknowledgement method sets to NOACK (with multiple option).

If requeue option sets in true then acknowledgement method sets to REJECT.

### Install (Lua>= 5.1)

1. First step

```sh
luarocks install amqp-client
```

2. Second Step

Replace /usr/local/share/lua/5.1/amqp/init.lua

```sh
git clone https://github.com/Zoviet/amqp-client
cd amqp-client
sudo cp init.lua /usr/local/share/lua/5.1/amqp/init.lua

```

or just copy ampq to your project.

### General usage

Consumer config example:

```
local args =  {}

args['x-dead-letter-exchange'] = 'test.retry'
args['x-message-ttl'] = 15000

local ctx = amqp:new({
    role = 'consumer',
    exchange = 'test',
    routing_key = 'test',
    queue = 'sms',
    ssl = false,
    user = 'guest',
    password = 'guest',
    no_ack = false,
    durable = true,
    auto_delete = false,
    consumer_tag = '',
    requeue = false, 
    multiple = false,
    exclusive = false,
    properties = {},
    arguments = args
})

```

Consumer loop example:

```
local ok, err
ok , err = ctx:connect(host, port)
if not ok then
    error('could not connect'..err)
end

ok, err = ctx:setup() -- because of this we need to use consume_loop()
if not ok then
    error('could not setup: '..err)
end

ok, err = ctx:prepare_to_consume() -- this has to be right after setup()
if not ok then
    error('could not prepare to consume: '..err)
end

local callback = function(f)
    print(f.frame.routing_key) 
end

ok, err = ctx:consume_loop(callback)
print(ok, err)

```

Message rejected when callback breaks with error:

```
local callback = function(f)
    print(f.frame.routing_key) 
    error('test error')
end
  
```

Logging:

```
local logger = require "amqp.logger"

logger.set_level(4) -- logging level for print : ERR   = 4  INFO  = 7  DEBUG = 8

```

Publisher loop example:

```
local ctx

local function connect()	
	ctx = amqp:new({
	    role = 'producer',
	    exchange = 'missed',
	    routing_key = 'missed',
	    ssl = false,
	    user = 'guest',
	    password = 'guest',
	    no_ack = false,
	    channel = 2,
	    durable = true,	
	    auto_delete = true,
	    consumer_tag = '',
	    exclusive = false,
	    properties = {}
	})	
	local ok1, err = ctx:connect(host, port)		
	if err then 
		log.error(err)
		ctx:close()
		wait(5)			
		connect()
	end
	local ok, setup_err = ctx:setup()
	if setup_err then
		log.error(setup_err) 	
		ctx:teardown()
		wait(15)
		connect()
	end		
	log.info('Connected')
	return ok
end

local function publish()
	while (true) do
		-- Do something
		local publish_ok, publish_err = ctx:publish(message)	
		if not publish_ok then 
			log.error('Publish error : '..publish_err)	
			ctx:teardown()
			ctx:close()	
			wait(1)
			connect()
		end
	end
end

if connect() then 
	publish() 
else
	ctx:close()
	error('Not connect to RabbitMq')
end

ctx:teardown()
	
```

This library is already tested with RabbitMQ and should work with any other AMQP 0.9.1 broker, follow this [Wiki Documentation](https://github.com/gsdenys/amqp-client/wiki) to know how to use this library.

## Develop

To facilitate to create the lua development environment, was created a [Lua Vagrant Image](https://github.com/gsdenys/lua-vagrant). Use it to gain time.

Case you already have your environment done, just clone this repository and start working. Other else, execute the following steps ([Vagrant](https://www.vagrantup.com) need to be installed)

### Prepare the Environmen

First of all you need to clone the [lua-vagrant](https://github.com/gsdenys/lua-vagrant) environment executing this command:

```sh
git clone https://github.com/gsdenys/vagrant-lua.git
cd vagrant-lua
```

As we'll install the [RabbitMQ](https://www.rabbitmq.com), is very interesting expose the ports to be accessed through the host, this way we can easilly access the administrator enviromnent in your browser. So, to to this, you need to locate the code below at the __Vagrantfile__

```sh
###
# add the network fowarded port here
# ex: config.vm.network "forwarded_port", guest: 15672, host: 8080
###
  
# config.vm.network "forwarded_port", guest: 8080, host: 8080
```
and insert the the next 2 lines after that:

```sh
config.vm.network "forwarded_port", guest: 15672, host: 8080
config.vm.network "forwarded_port", guest: 672, host: 672
```

So, let's clone this repository inside [lua-vagrant](https://github.com/gsdenys/lua-vagrant) project and start it.

```sh
git clone https://github.com/gsdenys/amqp-client.git

vagrant up #it takes a lot of time 
vagrant ssh
```
Now, you already are inside the [lua-vagrant](https://github.com/gsdenys/lua-vagrant) VM. Then you need to install the [RabbitMQ](https://www.rabbitmq.com).

```sh
#install RabbitMQ
wget https://bit.ly/2Ycybe8 -O install.sh
sh install.sh
```
Your environment is ready now. the amqp-client project is in the _/lua/amqp-client_ directory. Go to the project and start to use it.

```sh
#go to amqp-client source code
cd /lua/amqp-client
```

### Building

This library have some requirements shown below. If you're using the [Lua Vagrant Image](https://github.com/gsdenys/vagrant-lua) just take care about the busted lib, the others is already done.

1. LuaJIT >= 2.1 
2. busted 2.0 (Testing framework)
3. luabitop (if you are using lua 5.1)

To install busted lib >= 2.1 execute the following command:

```sh
luarocks install busted
```

you need to use the lua socket. Install it with this command:

```sh
luarocks install luasocket
```

After requirements solved, you can run the unit tests with busted using the following command:

```sh
busted
```

Wow you are ready to build the library executing following command::

```sh
luarocks make
```

the output should be like this:

    amqp-client 1.0.0-1 is now installed in /usr/local (license: Apache 2.0)

### Examples

The examples needs some dependencies that can be solved through the following commands:

```sh
luarocks install inspect
luarocks install lua-resty-uuid
luarocks install argparse
```

Beyond dependences, the examples depends on build of this library. Other way to execute the examples less building the library is import this library from luarocks. You can do this executing this command.

 ```sh
 luarocks install amqp-client
 ```   
