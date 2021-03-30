## Open Source Xerom Mining Pool forked from exlo84

![Miner's stats page](https://i.gyazo.com/f5361009debf4921a21f5fb3bd06b3b2.png)

### Features

**This pool is being further developed to provide an easy to use pool for Ethereum miners. This software is functional however an optimised release of the pool is expected soon. Testing and bug submissions are welcome!**

* Support for HTTP and Stratum mining
* Detailed block stats with luck percentage and full reward
* Failover geth instances: geth high availability built in
* Modern beautiful Ember.js frontend
* Separate stats for workers: can highlight timed-out workers so miners can perform maintenance of rigs
* JSON-API for stats

### Building on Linux -- highly recommend Ubuntu 16.04

Dependencies:

  * go = 1.14 or 1.15
  * geth or parity
  * redis-server >= 2.8.0
  * nodejs = 12 or 14 LTS
  * nginx

First of all let's get up to date and install the dependencies:

    sudo apt-get update && sudo apt-get dist-upgrade -y
    sudo apt-get install build-essential make git screen unzip curl nginx pkg-config nmap xterm screen tcl -y

Install GO:

    wget https://storage.googleapis.com/golang/go1.14.2.linux-amd64.tar.gz
    tar -xvf go1.14.2.linux-amd64.tar.gz
    rm go1.14.2.linux-amd64.tar.gz
    sudo mv go /usr/local
    export GOROOT=/usr/local/go
    export PATH=$GOPATH/bin:$GOROOT/bin:$PATH

Clone & compile:

    git config --global http.https://gopkg.in.followRedirects true
    git clone https://github.com/def670/XeroDreamPool.git
    cd XeroDreamPool
    chmod +x build/env.sh
    make
    **if your first make fails do not forget to "make clean" after you fix your issue

Installing Redis latest version

    wget http://download.redis.io/redis-stable.tar.gz
    tar xvzf redis-stable.tar.gz
    cd redis-stable
    make
    make test
    sudo make install
    
    sudo mkdir /etc/redis
    sudo cp ~/redis-stable/redis.conf /etc/redis
    sudo nano /etc/redis/redis.conf
    
     # Set supervised to systemd
      supervised systemd
     # Set the dir
      dir /var/lib/redis
      
Create a Redis systemd config file
     
     sudo nano /etc/systemd/system/redis.service
     
Add this to the redis system service file and save:

    [Unit]
    Description=Redis In-Memory Data Store
    After=network.target

    [Service]
    User=redis
    Group=redis
    ExecStart=/usr/local/bin/redis-server /etc/redis/redis.conf
    ExecStop=/usr/local/bin/redis-cli shutdown
    Restart=always

    [Install]
    WantedBy=multi-user.target
    
Create redis user/group/directorys

    sudo adduser --system --group --no-create-home redis
    sudo mkdir /var/lib/redis
    sudo chown redis:redis /var/lib/redis
    sudo chmod 770 /var/lib/redis
    
Using systemctl for pool operations: enable, start, restart, status, and stop commands
    
    sudo systemctl enable redis
    sudo systemctl start redis
    sudo systemctl restart redis
    sudo systemctl status redis
    sudo systemctl stop redis
    
    
### Install Geth

    cd ~
    wget -N https://github.com/xero-official/go-xerom/releases/download/2.1.0/geth-linux.zip
    unzip geth-linux.zip
    rm geth-linux.zip
    sudo mv geth /usr/local/bin/geth 

Make geth system service

    sudo nano /etc/systemd/system/geth.service
    
Copy the following

    [Unit]
    Description=Geth for Pool
    After=network-online.target
    
    [Service]
    ExecStart=/usr/local/bin/geth --rpc --rpcaddr 127.0.0.1 --rpcport 8545 --syncmode "fast" --etherbase <your-address> --mine --extradata "<your-pool>"
    User=<your-user-name>
    
    [Install]
    WantedBy=multi-user.target

These commands operate the geth system file

    sudo systemctl enable geth
    sudo systemctl start geth
    sudo systemctl restart geth
    sudo systemctl status geth
    sudo systemctl stop geth

Run console

    geth attach

Register pool account and open wallet for transaction. Registration is once if at all.  If you have an existing keystore file you can place it where it belongs, and just unlock the wallet as below.  Wallet has to be unlocked when geth is restarted.

    personal.newAccount()
    personal.unlockAccount(eth.accounts[0],"your-password",40000000)

### Set up pool

    mv config.example.json config.json
    nano config.json
    cp config.json build/bin/        <--dont forget to put your json files where the pool expects them
    
#######
additional suggested setup !!
#######
    I suggest you make your pool startup as 3 services instead of just the one laid out.
    I made 3 copys of the config.json file and named them:
    
      config.json      <--enable proxy, stratum, and API. disable payouts and unlocker
      payout.json      <--enable payouts.  disable proxy, stratum, API, and unlocker
      unlocker.json    <--enable unlocker.  disable proxy, stratum, API, and payouts
      
    then make additional startups as in the next step to call the unlocker and payouts.
    Running your pool like this means you can restart the pool or unlocker or payouts independantly
    with sudo systemctl restart *pool,payouts,unlocker*
    if you do it this way do not forget to move these config files to build/bin/
    
#######
end of addon
#######
    
Make pool system service

    sudo nano /etc/systemd/system/pool.service

Copy the following

    [Unit]
    Description=Xerom pool
    After=geth.target
    
    [Service]
    ExecStart=/home/<name>/XeroDreamPool/build/bin/open-ethereum-pool /home/<name>/XeroDreamPool/build/bin/config.json
    
    [Install]
    WantedBy=multi-user.target

Use sudo systemctl to call the pool just as above with redis or geth

    sudo systemctl enable pool
    sudo systemctl start pool
    sudo systemctl restart pool
    sudo systemctl status pool
    sudo systemctl stop pool

### Building Frontend

    cd www

Modify your configuration file

    nano ~/XeroDreamPool/www/config/environment.js

Create frontend

    cd ~/XeroDreamPool/www/
    
    sudo npm install -g ember-cli@2.9.1
    sudo npm install -g bower
    sudo chown -R $USER:$GROUP ~/.npm
    sudo chown -R $USER:$GROUP ~/.config
    npm install
    bower install
    npm i intl-format-cache
        intl-format-cache has issues with sammy007 open-ethereum-pool.  once npm has installed
        it do the following:
          cd www/node_modules
          wget https://github.com/sammy007/open-ethereum-pool/files/3618316/intl-format-cache.zip
          rm intl-fomat-cache
          unzip intl-format-cache.zip
          cd intl-format-cache
          npm install
          then drop back to www/ and build pool
          
    ./build.sh


Configure nginx to serve API on <code>/api</code> subdirectory.
Configure nginx to serve <code>www/dist</code> as static website.

#### Serving API using nginx

Edit this

    sudo nano /etc/nginx/sites-available/default

Delete everything in the file and replace it with the text below.
Be sure to change with your info

    upstream api {
            server 127.0.0.1:8080;
    }
    
    server {
      listen 80 default_server;
      listen [::]:80 default_server;
      root /home/<name>/XeroDreamPool/www/dist;
     
     index index.html index.htm index.nginx-debian.html;
     
    server_name _;
     
    location / {
            try_files $uri $uri/ =404;
            }
      
    location /api {
            proxy_pass http://api;
            }
    }
    
Save and close

Restart nginx

    sudo service nginx restart

### Notes

* Unlocking and payouts are sequential, 1st tx go, 2nd waiting for 1st to confirm and so on. You can disable that in code. Carefully read `docs/PAYOUTS.md`.
* Also, keep in mind that **unlocking and payouts will halt in case of backend or node RPC errors**. In that case check everything and restart.  **if you follow the multiple json configs then you can restart just the affected portion.
* You must restart module if you see errors with the word *suspended*.
* Don't run payouts and unlocker modules as part of mining node. Create separate configs for both, launch independently and make sure you have a single instance of each module running. 
* If `poolFeeAddress` is not specified all pool profit will remain on coinbase address. If it specified, make sure to periodically send some dust back required for payments.

### Credits

Made by sammy007. Licensed under GPLv3.

Graphing and style added by Exlo84.

#### Contributors

[Alex Leverington](https://github.com/subtly)
[Primate411](https://github.com/Primate411/)
[Exlo84](https://github.com/Exlo84/)
[def670](https://github.com/def670/)
