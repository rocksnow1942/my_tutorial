<!-- 
### Emphasis
*Italic* ; **Bold** ; ***Bold and Italic*** ; ~~Scratch~~
### List items
1. First ordered list item
* Unordered list can use asterisks --> 
# Setup my x201 as download server

* ### [Remove bookmark folder in nautilus](https://askubuntu.com/questions/79150/how-to-remove-bookmarks-from-the-nautilus-sidebar)
    ```bash
    sudo nano /etc/xdg/user-dirs.defaults
    # then comment out unwanted links 
    nano .config/user-dirs.dirs 
    # then comment out unwanted links 
    # afterwards, can dellete favirotes links in nautilus.
    ```

* ### [Prevent laptop sleep](https://askubuntu.com/questions/15520/how-can-i-tell-ubuntu-to-do-nothing-when-i-close-my-laptop-lid)    
    ```bash
    sudo nano /etc/systemd/logind.conf
    # change HandleLidSwitch=lock
    sudo systemctl restart systemd-logind
    # this log off and restart systemd daemon
    ```
* ### Install python and git
    ```bash 
    sudo apt-get install build-essential 
    # then download python 
    # install anaconda python 
    bash '/path/to/anaconda.sh'
    sudo apt install git
    ```
* ### [Install nodejs and NPM](https://github.com/nodesource/distributions)
    ```bash 
    curl -sL https://deb.nodesource.com/setup_13.x | sudo -E bash -
    sudo apt-get install -y nodejs
    sudo npm install -g n # insall n package. 
    ```

* ### Openssh setup 
    ```bash 
    sudo apt update 
    sudo apt-get upgrade 
    sudo apt install openssh-server 
    sudo systemctl restart ssh 
    sudo systemctl status ssh 
    ```
    Then from my other computer:
    ```bash 
    # copy ssh pubkey to server; then I can connect with key
    ssh-copy-id <user>@thinkpad.lan
    ```
    After that I can conncet with ssh and key. 

    I add the following edits to `/etc/ssh/sshd_config`;
    also create banner at `/etc/issue`
    ```bash 
    AllowUsers '###'
    PermitRootLogin no 
    StrictModes yes
    MaxAuthTries 6
    MaxSessions 10
    ChallengeResponseAuthentication no
    UsePAM no
    X11Forwarding yes
    PrintMotd no
    Banner /etc/issue
    PasswordAuthentication no
    Subsystem       sftp    /usr/lib/openssh/sftp-server
    AcceptEnv LANG LC_*
    ```
    Allow ssh in ufw `sudo ufw allow ssh`

* ### [Samba Share folder](https://ubuntu.com/tutorials/install-and-configure-samba#1-overview) 
    ```bash 
    sudo apt install samba 
    mkdir /home/<uysername>/sambashare 
    sudo nano /etc/samba/smb.conf # edit config 
    # under [global] add the following to restrict access
    restrict anonymous = 2 
    # add to the bottom:
    [sambashare]
    comment = Thinkpad Samba share
    path = /home/<username>/sambashare
    read only = no
    browsable = yes
    # then restart 
    sudo service smbd restart 
    sudo ufw allow samba
    ```
    Then setup samba share user and password. 
    ```bash 
    sudo smbpasswd -a <username>
    ```

* ### [Setup Download Server with aria2](https://medium.com/@sajithneyo/remote-download-server-in-linux-using-aria2-5bd3ee1a54b2)
    ```bash 
    sudo apt install aria2 
    # inside home dir
    mkdir aria2 
    mkdir ./aria2/conf 
    mkdir ./aria2/ses
    touch ./aria2/ses/aria2.session # need to create this file for aria2. 
    ```
    Inside `conf` folder, creat `aria.conf` file with following:
    ```bash
    ##files
    dir=/home/<user>/sambashare
    file-allocation=falloc
    continue=true
    daemon=true
    disk-cache=32M

    ##logging
    log=/home/<user>/aria2/aria2.log
    console-log-level=warn
    log-level=notice

    ##downloads
    max-concurrent-downloads=5
    max-connection-per-server=5
    min-split-size=20M
    split=4
    disable-ipv6=true
    max-overall-upload-limit=2048

    ##sessions
    force-save=true
    input-file=/home/<user>/aria2/ses/aria2.session
    save-session=/home/<user>/aria2/ses/aria2.session
    save-session-interval=10

    ##security
    http-auth-challenge=true
    check-certificate=false
    enable-rpc=true
    rpc-listen-all=true
    rpc-secret=<TOKEN> 
    #token will be used for webUI access, 
    # use 'sudo openssl rand -base64 32' to generate token.

    ##ports
    rpc-listen-port=6800

    ##others
    summary-interval=120
    enable-dht=true

    ##times
    timeout=600
    retry-wait=30
    max-tries=50
    ```
    Add aria2 to systemd service:
    ```bash 
    sudo nano /lib/systemd/system/aria2.service 
    # add following content:
        [Unit]
        Description=Aria2c download manager
        Requires=network.target
        After=dhcpcd.service
        [Service]
        Type=forking
        User=<user>
        RemainAfterExit=yes
        ExecStart=/usr/bin/aria2c --conf-path=/home/<user>/aria2/conf/aria.conf
        ExecReload=/usr/bin/kill -HUP $MAINPID
        ExecStop=/usr/bin/kill -s STOP $MAINPID
        RestartSec=1min
        Restart=on-failure
        [Install]
        WantedBy=multi-user.target
    # start service:
    sudo service aria2 start 
    sudo service aria2 status 
    sudo systemctl enable aria2
    ```
    
    Install Apache2 server:
    ```bash
    sudo apt-get install apache2
    sudo ufw app list 
    sudo ufw enable Apache 
    sudo systemctl enable apache2
    ```
    Get WebUI for aria2 
    ```bash 
    cd /var/www/html
    sudo git clone https://github.com/ziahamza/webui-aria2.git
    sudo cp -r webui-aria2/docs ./aria2
    sudo ufw allow 6800
    ```


* ### UFW configure 
    Configure firewall 
    ```
    sudo ufw enable 
    sudo ufw status
    ```

* ### [Power management](https://linrunner.de/en/tlp/docs/tlp-linux-advanced-power-management.html)
    ```bash 
    sudo add-apt-repository ppa:linrunner/tlp
    sudo apt update
    sudo apt install tlp tlp-rdw
    tlp-stat -b # show that x201 need to install smapi
    sudo apt install tp-smapi-dkms
    sudo tlp start # start / restart tlp 
    sudo tlp-stat -s # show tlp status 
    sudo tlp [start|ac|bat]    
    sudo tlp-stat -[b|c|t] # show bat|config|thermal stat 
    sudo tlp setcharge [ START_THRESH STOP_THRESH [ BAT0 | BAT1 ] ] # set charge threshold 
    ```
* ### [Install Pi-Hole](https://linuxincluded.com/install-pi-hole-on-ubuntu/)
    ```bash 
    # start install pihole:
    curl -sSL https://install.pi-hole.net | bash
    pihole -a -p # to change password
    pihole -d # to show diagnostic. 
    pihole -r # to configure pihole again. 
    # Choose An Interface: use wlan for wifi pi, use eth for wired connection.
    pihole -t # tail dns query 
    ```
    Then set DNS to pihole ip to use. 
    web interface through `pi.hole/admin` or `thinkpad.lan/admin` 
    This web interface is using `lighttpd`
    The http server config can be edit from:
    `/etc/lighttpd/lighttpd.conf` 

    To serve the admin page with apache2, nee to install php.
    
    To install php for apache2:
    ```bash
    sudo add-apt-repository ppa:ondrej/php
    sudo apt-get update
    sudo apt-get install php7.0
    ```
    After this, apache2 should serve php normally. 
    ```bash 
    # disable lighttp
    sudo systemctl disable lighttpd
    sudo systemctl enable apache2 
    sudo service apache2 restart
    ```
    To set DHCP using Pi-Hole, Goto PI Hole settings -> DHPC tab, enable DHCP server. 
    
    Also need to allow DHCP in ufw:
    ```
    sudo ufw allow from any to any port 67 proto udp
    ```

    On google wifi app, set DNS to pihole ip address, secondary dns use google dns. set LAN DHCP pool to ip of pihole. 
    Then reserve IP of pihole. Thus DHCP of google wifi will not assign IP, and all ips are assigened from pihole.

    **Custom Host through Pi-Hole**
    
    Edit `/etc/hosts` to add custom host name.
    ```bash
    192.168.86.9    thinkpad.lan
    ```
    Then run [`pihole restartdns`](https://docs.pi-hole.net/core/pihole-command/) to restart dns service.

* ### [Install Jellyfin](https://tipsmake.com/how-to-set-up-media-server-at-home-with-jellyfin-on-ubuntu)
    ```bash 
    sudo apt install apt-transport-https
    wget -O - https://repo.jellyfin.org/jellyfin_team.gpg.key | sudo apt-key add -
    echo "deb [arch=$( dpkg --print-architecture )] https://repo.jellyfin.org/$( awk -F'=' '/^ID=/{ print $NF }' /etc/os-release ) $( awk -F'=' '/^VERSION_CODENAME=/{ print $NF }' /etc/os-release ) main" | sudo tee /etc/apt/sources.list.d/jellyfin.list
    sudo apt update
    sudo apt install jellyfin
    # manage jellyfin with 
    sudo systemctl {action} jellyfin.service # or
    sudo service jellyfin {action}
    sudo ufw allow 8096 # jellyfin use port 8096
    ```
    Then can access jellyfin from: `<pi/hole>:8096`

    Subtitle support: Install OpenSubtitle in plugins.

    **[Add reverse proxy for Jellyfin](https://www.linode.com/docs/applications/media-servers/how-to-install-jellyfin/)**

    ```bash
    sudo a2enmod proxy proxy_http
    sudo nano /etc/apache2/sites-available/movie.pi.hole.conf
    # add following configuration for proxy.
    <VirtualHost *:80>
    ServerName movie.pi.hole
    ErrorLog /var/log/apache2/jellyfin-error.log
    CustomLog /var/log/apache2/jellyfin-access.log combined
    ProxyPreserveHost On
    ProxyPass "/" "http://127.0.0.1:8096/"
    ProxyPassReverse "/" "http://127.0.0.1:8096/"
    </VirtualHost>
    # enable new website by:
    sudo a2ensite movie.pi.hole.conf 
    # restart apache 
    sudo systemctl restart apache2 
    # add movie.pi.hole to Pi-Hole host:
    sudo nano /etc/hosts
    # Add ip and domain movie.pi.hole then restart host 
    pihole restartdns 
    ```
    This will allow access jellyfin from *`movie.pi.hole`*. To access from pi.hole/movie, use redirect.

    ```bash 
    # edit default apache settings:
    sudo nano /etc/apache2/sites-available/000-default.conf
    # add this line:
    Redirect "/movie" "http://movie.pi.hole"
    ```

* ### [Install Webmin](https://linuxize.com/post/how-to-install-webmin-on-ubuntu-18-04/)
    ```bash 
    sudo apt update
    sudo apt install software-properties-common apt-transport-https wget
    # import Webmin GPG key
    wget -q http://www.webmin.com/jcameron-key.asc -O- | sudo apt-key add -
    # enable webmin repo
    sudo add-apt-repository "deb [arch=amd64] http://download.webmin.com/download/repository sarge contrib"
    sudo apt install webmin
    ```
    [Setup host in apache for webmin:](https://vitux.com/install-and-configure-webmin-on-ubuntu/)
    ```bash 
    sudo nano /etc/apache2/sites-available/admin.pi.hole.conf
    # similar to jellyfin, add following
    <VirtualHost *:80>
    ServerName admin.pi.hole
    ProxyPass / http://localhost:10000/
    ProxyPassReverse / http://localhost:10000/
    </VirtualHost>
    # edit default apache settings:
    sudo nano /etc/apache2/sites-available/000-default.conf
    # add this line:
    Redirect "/admin" "http://admin.pi.hole"
    
    sudo a2ensite admin.pi.hole
    sudo systemctl reload apache2
    # then similar to above, add admin.pi.hole to pihole host list

    # Stop webmin from using TLS/SSL. 
    sudo nano /etc/webmin/miniserv.conf
    # change:
    ssl=0

    # Add your domain name to the list of allowed domains
    sudo nano /etc/webmin/config
    # add to the end:
    referers=admin.pi.hole
    # restart webmin
    sudo systemctl restart webmin
    ```
    Then access webmin from admin.pi.hole




    
