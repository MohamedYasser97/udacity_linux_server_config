# Linux Server Configuration Project
In this project, I've taken my previous [Item Catalog](https://github.com/MohamedYasser97/udacity_item_catalog) project and hosted it on a bare-bone VPS(LightSail) running Ubuntu.

In this file I will explain how Udacity's reviewer can access my LightSail instance and I will also walk through the process of building this project.

## List of contents
  1. [Info needed for Udacity's reviewer.](#step1)
  2. [Creating a LightSail instance](#step2)
  3. [Using SSH for the main 'ubuntu' user.](#step3)
  4. [Making sure every package is up to date.](#step4)
  5. [Creating the 'grader' user and granting them root access and SSH.](#step5)
  6. [Setting the timezone to UTC and configuring the firewall.](#step6)
  7. [Creating and configuring the database and its limited user.](#step7)
  8. [Installing Apache.](#step8)
  9. [Cloning the item catalog code, setting up its virtual environment and some code changes.](#step9)
  10. [Setting up a virtual host.](#step10)
  11. [Configuring the WSGI application.](#step11)
  12. [Final touches.](#step12)
  
 ---
 
 <a name="step1"></a>
 ## 1) Info needed for Udacity's reviewer
 
 The reviewer can access my web app through its IP address (`54.93.101.209`) or this URL: 
 
 http://54.93.101.209/
 
 But in order to evaluate the project you have to SSH into my LightSail instance through port `2200` with the username `grader`.
 
 This can be achieved by executing the following command:
 
 `ssh grader@54.93.101.209 -p 2200 -i grader.rsa`
 
 You may have noticed that `grader.rsa` isn't handed in this file because it's a __private__ key so it will be handed in the submission notes.
 
 __* The `grader` and `catalog`'s UNIX passwords will also be handed through the submission notes!*__
 
 In the next sections I will explain how every step was taken __including__ setting up the `grader`'s SSH keys.
 
 ---
 
 <a name="step2"></a>
 ## 2) Creating a LightSail instance
 
 Of course Amazon's LightSail isn't a must but it's what the instructors recommended and it's very easy to set up.
 
 I won't go through billing options as it's completely up to you but what we will need is to set up an instance that runs __Ubuntu__ under the __OS only__ option, we __DO NOT__ want our instance to come with any applications.
 
 After creating an instance we will be redirected to the instance's control panel where we would find the IP address and other important 'Networking' settings we will need later.
 
 ---
 
<a name="step3"></a>
 ## 3) Using SSH for the main 'ubuntu' user
 
 In your instance's control panel you'll find a big nice orange button that says "Connect using SSH". The aim of this section is to achieve exactly what this button does (feel free to try it one last time!) but from any terminal no just the browser.
 
 To do so, we need to get our __SSH Keys__ for this instance. Fortunately they can be found in your __"Account"__ settings under a submenu called __"SSH Keys"__.
 
 Click on download and feel free to rename it to anything but with the extension `.rsa` instead of this `.pem`.
 
 The public keys are already saved on the instance's disk and now we have our private key on our local machine so we can connect to the instance through the terminal with this command:
 
 `ssh ubuntu@YOUR_INSTANCE's_IP -p 2200 -i YOUR_PRIVATE_KEY.rsa`
 
 __*Resources__: Udacity's video lessons.
 
 ---
 
 <a name="step4"></a>
 ## 4) Making sure every package is up to date
 
 You may think that just running 
 
 `sudo apt-get update`
 
 `sudo apt-get upgrade`
 
 is enough. They __are__ needed but you __also__ need to run
 
 `sudo apt-get dist-upgrade`
 
 because some packages need to be entirely removed or installed, and not just upgrading the package's version, so it's worth mentioning.
 
 __*Resources__: StackOverflow's first google search result
 
 ---
 
 <a name="step5"></a>
 ## 5) Creating the 'grader' user and granting them root access and SSH
 
 After logging in with the `ubuntu` user through SSH. We can simply create the new user `grader` with the command
 
 `sudo adduser grader`
 
 You will be prompted to enter your password and some other optional info.
 
 Now we want to grant `grader` __sudo__ access. You can either add a `grader` file in the `sudoers.d` directory or just use `sudo visudo` like I did and both will work the same way.
 
 So after executing `sudo visudo` we will scroll through the file until we find the line that dictates the root user's permission so that we can add `grader`'s permissions under it. It should look like this:
 
 `root    ALL=(ALL:ALL) ALL`
 
 After we add our new entry it should look like this:
 
 ```bash
  root    ALL=(ALL:ALL) ALL
  grader  ALL=(ALL:ALL) ALL
  ```
  
  Now save and exit (`CTRL+X` then `Y`) and now our `grader` has SUDO access!
  
  Now we want to allow the `grader` to be able to login remotely to their account through SSH. For our `ubuntu` user, Amazon provided us with both public and private keys but now we need to generate our own.
  
  On your __LOCAL__ machine run the following command and select the directory to save the two output files in:
  
  `ssh-keygen`
  
  Now you should have __two__ files, `NAME` and `NAME.pub`, the first is the `grader`'s private key and the other is the public key. __Only you and Udacity's reviewer__ should have access to the private key. Now we'll copy the public key's contents and login with `ubuntu` as mentioned before and switch users to `grader` by executing the following:
  
  `su grader`
  
  Proceed to create a new directory `mkdir ~/.ssh` and create a new file `sudo nano ~/.ssh/authorized_keys`, paste the public key in it and then save and exit.
  
  As a security measure we will alter the directory and file's permissions by executing:
  
  ```bash
  sudo chmod 700 .ssh
  sudo chmod 644 .ssh/authorized_keys
  ```
  
  Now to fulfill Udacity's requirement, open the file `/etc/ssh/sshd_config` and check that this field is set like this `PasswordAuthentication no`.
  
  At this point you should be able to login with `grader` through SSH!
  
  __*Keep logged in as `grader` for the next steps.*__
  
  __*Resources__: Udacity's video lessons.
  
  ---
  
  <a name="step6"></a>
 ## 6) Setting the timezone to UTC and configuring the firewall
 
 Setting the timezone can be easily done by executing
 
 ```sudo timedatectl set-timezone UTC```
 
 To configure the firewall to the required ports and rules, you can simply execute the following commands in order:
 
 ```
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 2200/tcp
sudo ufw allow 123/udp
sudo ufw allow www
sudo ufw deny 22
 ```
 
 and then enable the firewall by executing `sudo ufw enable`.
 
 We have to apply the following changes to LightSail's control panel too. From the `Networking` section we should delete the firewall's `SSH 22` entry and enter __two__ new entries: `CUSTOM TCP 2200` and `CUSTOM UDP 123`. 
 
 *__Now our instance is only accessible through the terminal, no more browsers!__*
 
 __*Resources__: Udacity's video lessons and [this link](https://linuxize.com/post/how-to-set-or-change-timezone-on-ubuntu-18-04/).
 
 ---
 
 <a name="step7"></a>
 ## 7) Creating and configuring the database and its limited user
 
 While still being logged in as `grader`, install PostgreSQL by executing `sudo apt-get install postgresql`. Luckily, PostgreSQL doesn't allow remote connections by default.
 
 Now in order to create the `catalog` user we first need to switch to the `postgres` user that was created for us by executing `sudo su postgres`. Now we will create the `catalog` user's database __role restrictions__ before we create `catalog` itself.
 
 You may ask, why don't we just use the `postgres` user? That's because it has a lot of authority which we can check with `\du` after running `psql`. We want our `catalog` user to only be able to create tables. We can do this by running the following commands inside `psql`:
 
 ```SQL
 CREATE ROLE catalog WITH LOGIN PASSWORD 'YOUR_OWN_PASSWORD';
 ALTER ROLE catalog CREATEDB;
 ```
 Now we just have to switch back to `grader` and create the `catalog` user just like we created `grader` (__PASSWORD WILL BE HANDED IN SUBMISSION NOTES!__).
 
 You may have guessed it, we would want to give `catalog` root access too (I have explained earlier how to do that).
 
 After giving `catalog` root access, we will switch back to `catalog` and use its only role, execute this command: `createdb catalog`.
 
 __*Resources__: Udacity's video lessons but the main credit goes to [this guide](https://www.digitalocean.com/community/tutorials/how-to-secure-postgresql-on-an-ubuntu-vps).
 
 ---
 
 <a name="step8"></a>
 ## 8) Installing Apache
 
 After switching back to `grader`, we wil install Apache by executing `sudo apt-get install apache2`. And since we're running __Python 3__, we will also install this package `sudo apt-get install libapache2-mod-wsgi-py3`. Then we have to enable it by running `sudo a2enmod wsgi`.
 
 __*Resources__: Random google search results.
 
 ---
 
 <a name="step9"></a>
 ## 9) Cloning the item catalog code, setting up its virtual environment and some code changes
 
 We need to clone the __Item Catalog__'s code inside this exact directory `/var/www/catalog/`. In my case it would be `/var/www/catalog/udacity_item_project`. Change the `catalog/` directory's owner to `grader`.
 
 Using a virtual environment is the safest way when in the context of hosting many apps from the same machine. We would essentially create "virtual" Python binaries and libraries that don't conflict with the same files found on the real environment.
 
 We can do so by installing PIP first (`sudo apt-get install python3-pip`). Then we install the virtual environment itself (`sudo apt-get install python-virtualenv`). 
 
 Then we can initialize the environment from the project's directory `sudo virtualenv -p python3 venv3`. Change `venv3`'s ownership to `grader` too.
 
 Finally, activate the virtual environment with `. venv3/bin/activate` and proceed to install all your projects dependencies (I will leave this up to you since I don't know if you have a `requirements.txt` file like me or not, if you have, just execute `sudo pip install -r requirements.txt`).
 
 Now to the __code changes__!
 
 __First__ of all change your main python file to `__init__.py`.
 
 __Second__, remove every line that has this line:
 
 ```python
 engine = create_engine("sqlite:///catalog.db")
 ```
 
 to this line:
 
 ```python
 engine = create_engine('postgresql://catalog:PASSWORD@localhost/catalog')
 ```
 
 __Third__, in the `__init__.py` file, edit the this line from `app.run(host="0.0.0.0", port=8000, debug=True)` to just `app.run()`.
 
 __Lastly__, replace all instances of `"client_secret.json"` to `"/var/www/catalog/udacity_item_catalog/clent_secret.json"`.
 
 __*Resources__: Udacity's question forums, [this guide](https://www.geeksforgeeks.org/python-virtual-environment/), and LOTS AND LOTS OF TRIAL AND ERROR.
 
 ---
 
 <a name="step10"></a>
 ## 10) Setting up a virtual host
 
 To recap, we have our code ready and we have Apache running, but we need to tell Apache that requests coming to it should invoke "some" python code, or rather, the __Item Catalog__ code. That's when we'll need to configure the "Virtual Host".
 
 In order to tell Apache to use Python 3, we need to add this line to `/etc/apache2/mods-enabled/wsgi.conf`:
 
 ```
 WSGIPythonPath /var/www/catalog/udacity_item_catalog/venv3/lib/python3.5/site-packages
 ```
 
 Then we have to create the file `/etc/apache2/sites-available/item_catalog.conf` and write this inside it:
 
 ```
 <VirtualHost *:80>
    ServerName PASTE_YOUR_OWN_IP_HERE
    WSGIScriptAlias / /var/www/catalog/item_catalog.wsgi
    <Directory /var/www/catalog/udacity_item_catalog/>
    	Order allow,deny
  	  Allow from all
    </Directory>
    Alias /static /var/www/catalog/udacity_item_catalog/static
    <Directory /var/www/catalog/udacity_item_catalog/static/>
  	  Order allow,deny
  	  Allow from all
    </Directory>
    ErrorLog ${APACHE_LOG_DIR}/error.log
    LogLevel warn
    CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
 ```
 
 Then to activate this file we just created we execute `sudo a2ensite catalog` and then reload Apache, `sudo service apache2 reload`.
 
 __*Resources__: Random StackOverflow answer.
 
 ---
 
 <a name="step11"></a>
 ## 11) Configuring the WSGI application
 
 While we were creating our virtual host in [Step 10](#step10), we mentioned a file called `/var/www/catalog/item_catalog.wsgi`, now we will create it.
 
 We should write this inside it:
 
 ```python
activate_this = '/var/www/catalog/udacity_item_catalog/venv3/bin/activate_this.py'
with open(activate_this) as file_:
    exec(file_.read(), dict(__file__=activate_this))

#!/usr/bin/python
import sys
import logging
logging.basicConfig(stream=sys.stderr)
sys.path.insert(0, "/var/www/catalog/udacity_item_catalog/")
sys.path.insert(1, "/var/www/catalog/")

from catalog import app as application
application.secret_key = "YOUR_OWN_APPS_SECRET_PHRASE!"
 ```
 
 Then save, exit and restart Apache `sudo service apache2 restart`.
 
 __*Resources__: Flask's documentation.
 
 ---
 
 <a name="step12"></a>
 ## 12) Final touches
 
 *__This section is a bit specific to my own project, so it shouldn't strictly apply on your situation__*
 
 In my case, I created the actual Item Catalog's tables by executing `db_setup.py` and populated them using `db_populate.py`. I also updated the `client_secret.json` file after adding my IP to Google's credentials settings page. Unfortunately, Logging in with Google in my app is __NOT POSSIBLE__ because Google forces __HTTPS__ while the best I can do in my situation is use the `xip.io` wildcard to bypass Google's form and be able to submit my IP as an authorized Javascript origin. But to login in my app I __must__ use HTTPS which is outside the scope of this project.
 
 
 At this point however, I have successfully hosted my Item Catalog app and it can be accessed through the link in [Step 1](#step1).
