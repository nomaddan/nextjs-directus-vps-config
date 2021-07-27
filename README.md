# 1. Pre-requisites

1. VPS, ideally Ubuntu LTS distribution (I recommend OVH)
2. Domain DNS `A` records pointing to your server IP address
3. Next.js (or any other Node.js app) github repo

# 2. Initial set-up on VPS

1. Access your VPS through SSH
   ```
   ssh root@vps_ip
   ```
   You should have received an e-mail from VPS provider with login details
2. [Create new user with admin privileges](https://www.digitalocean.com/community/tutorials/initial-server-setup-with-ubuntu-20-04)
3. `exit` from the root account
4. SSH with your new user
   ```
   ssh new_user@vps_ip
   ```
5. [Generate SSH token for Github](https://docs.github.com/en/github/authenticating-to-github/connecting-to-github-with-ssh/generating-a-new-ssh-key-and-adding-it-to-the-ssh-agent)
6. [Add it to your GitHub account settings](https://docs.github.com/en/github/authenticating-to-github/connecting-to-github-with-ssh/adding-a-new-ssh-key-to-your-github-account)
   a. You can use `cat ~/.ssh/id_ed25519.pub` command to copy the SSH public key from VPS

# 3. Front-end app and PM2 set-up

1. Create `apps` folder with

   ```
   cd ~ && mkdir apps && cd apps
   ```

2. Install [NVM](https://github.com/nvm-sh/nvm), then install Node.js
   ```
   nvm install node
   ```
3. Install [PM2](https://pm2.keymetrics.io/docs/usage/quick-start/)
   ```
   npm install pm2@latest -g
   ```
4. Clone your Next.js repo
   ```
   git clone git@github.com:username/repo.git
   ```
   a. `cd` into it then
   ```
   npm install && npm run build
   ```
5. Create PM2 ecosystem config in `~/apps` with
   ```
   pm2 ecosystem
   rm ecosystem.config.js
   touch ecosystem.config.js
   ```
6. Open ecosystem file
   ```
   nano ecosystem.config.js
   ```
   and populate it with:
   ```
   module.exports = {
     apps: [
       {
         name: 'next',
         cwd: '/home/YOUR_VPS_USERNAME/apps/YOUR_REPO_NAME/',
         script: 'npm',
         args: 'start',
         exec_interpreter: '/home/YOUR_VPS_USERNAME/.nvm/versions/node/NODE_VERSION/bin/node',
         env: {
           // here you can provide ENV variables
           NEXT_PUBLIC_TEST: 'NEXT_PUBLIC_TEST',
         },
       },
     ],
   };
   ```
   remember to change `YOUR_VPS_USERNAME`, `YOUR_REPO_NAME`, `NODE_VERSION` (check with `node -v`)
7. Start PM2 ecosystem
   ```
   pm2 start ecosystem.config.js
   ```
8. Check if your app is running
   ```
   curl http://localhost:3000
   ```
   If not, then use
   ```
   pm2 logs --lines=9000
   ```
   to debug. Fix errors, then
   ```
   pm2 kill && pm2 flush
   ```
   to kill all processess and delete the the old logs. Then start ecosystem again
   ```
   pm2 start ecosystem.config.js
   ```
9. Configure PM2 to restart your apps automatically and let it start on system reboot. First run

   ```
   pm2 startup systemd
   ```

   and then run the command from the output of above command 10. Save your PM2 environment

   ```
   pm2 save
   ```

10. Start the PM2 service

    ```
    sudo systemctl start pm2-YOUR_USERNAME
    ```

    and reboot the system

    ```
    sudo reboot
    ```

    Then connect to VPS through SSH again

11. Run
    ```
    pm2 monit
    ```
    profit 🎉
12. Useful PM2 commands
    a. `pm2 list` - lists the apps managed by PM2
    b. `pm2 restart app_name` - restarts the app
    c. `pm2 info app_name` - gets info about app
    d. `pm2 monit` - process monitor displaying resources usage and their statuses

# Nginx set-up

1. Install Nginx

   ```
   sudo apt update && sudo apt install nginx
   ```

2. Configure firewall
   a. Check available apps

   ```
   sudo ufw app list
   ```

   b. Add Nginx HTTP app

   ```
   sudo ufw allow 'Nginx HTTP'
   ```

   c. Verify firewall status

   ```
   sudo ufw status
   ```

   d. Check if Nginx service is running

   ```
   systemctl status nginx
   ```

   e. Open your site VPS address in the browser. You can check the IP addres via

   ```
   curl -4 icanhazip.com
   ```

   You should see the default Nginx page 🎉

   f. Create configuration file for Nginx site

   ```
   sudo nano /etc/nginx/sites-available/YOUR_DOMAIN.com
   ```

   with:

   ```
   server {
   listen 80;
   listen [::]:80;

        server_name YOUR_DOMAIN.COM www.YOUR_DOMAIN.COM;

        location / {
            proxy_pass http://localhost:3000;
            proxy_http_version 1.1;
            proxy_set_header X-Forwarded-Host $host;
            proxy_set_header X-Forwarded-Server $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection "Upgrade";
            proxy_set_header Host $http_host;
            proxy_pass_request_headers on;
        }

   }

   ```

   remember to change `YOUR_DOMAIN.com`

3. Create a symlink from `sites-available` to `sites-enabled` using

   ```
   sudo ln -s /etc/nginx/sites-available/YOUR_DOMAIN.com /etc/nginx/sites-enabled/
   ```

4. Make sure there are no errors in Nginx config

   ```
   sudo nginx -t
   ```

5. Configure Nginx start up on system reboot

   ```
   sudo systemctl enable nginx
   ```

6. Open Nginx config file

   ```
   sudo nano /etc/nginx/nginx.conf
   ```

   and uncomment the line:

   ```
   server_names_hash_bucket_size 64;
   ```

7. Restart Nginx

   ```
   sudo systemctl restart nginx
   ```

   and visit `http://YOUR_DOMAIN.com` 🎉

# Secure your Nginx site with Let's Encrypt using [Certbot](https://certbot.eff.org/)

1. Update firewall settings to allow only HTTPS

   ```
   sudo ufw allow 'Nginx Full'
   sudo ufw delete allow 'Nginx HTTP'
   ```

2. [Install Certbot](https://certbot.eff.org/lets-encrypt/ubuntufocal-nginx)

   ```
   sudo snap install core
   sudo snap refresh core
   sudo snap install --classic certbot
   ```

3. Ensure Certbot installed correctly

   ```
   sudo ln -s /snap/bin/certbot /usr/bin/certbot
   ```

4. Obtain certificate

   ```
   sudo certbot --nginx -d YOUR_DOMAIN.com -d www.YOUR_DOMAIN.com
   ```

5. Check renewal process status

   ```
   sudo systemctl status certbot.timer
   ```

6. Manualy test the renewal dry-run

   ```
   sudo certbot renew --dry-run
   ```

7. Visit `https://YOUR_DOMAIN.com` 🎉

# Resources

- https://www.digitalocean.com/community/tutorials/initial-server-setup-with-ubuntu-20-04
- https://docs.github.com/en/github/authenticating-to-github/connecting-to-github-with-ssh/generating-a-new-ssh-key-and-adding-it-to-the-ssh-agent
- https://docs.github.com/en/github/authenticating-to-github/connecting-to-github-with-ssh/adding-a-new-ssh-key-to-your-github-account
- https://www.digitalocean.com/community/tutorials/how-to-set-up-a-node-js-application-for-production-on-ubuntu-20-04
- https://www.digitalocean.com/community/tutorials/how-to-install-nginx-on-ubuntu-20-04

# todo

1. add directus app in the next steps to ecosystem, add ssl

```

{
name: 'cms',
cwd: '~/apps/cms/',
script: 'npm',
args: 'start',
exec_interpreter: '~/.nvm/versions/node/NODE_VERSION/bin/node',
},

```
