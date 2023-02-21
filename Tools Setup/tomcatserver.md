https://linuxize.com/post/how-to-install-tomcat-10-on-ubuntu-22-04/

Installing Java
    Tomcat 10 requires Java version 11 or later to be installed on the system. We’ll install OpenJDK 11 , the open-source implementation of the Java Platform.

    Execute the following commands as root or user with sudo privileges to update the packages index and install the OpenJDK 11 JDK package:

sudo apt update
sudo apt install openjdk-11-jdk

Creating a System User
    Running Tomcat under the root user is a security concern and can be dangerous. The following command creates a new system user and group with home directory /opt/tomcat that will run the Tomcat service:

sudo useradd -m -U -d /opt/tomcat -s /bin/false tomcat

Downloading Tomcat
    Tomcat binary distribution can be downloaded from the Tomcat software downloads page .

    At the time of writing, the latest Tomcat version is 10.1.4. Before continuing with the next step, visit the Tomcat 10 download page and check if a newer version is available.

    Download the Tomcat zip file to the /tmp directory using the wget command:

VERSION=10.1.5
wget https://www-eu.apache.org/dist/tomcat/tomcat-10/v${VERSION}/bin/apache-tomcat-${VERSION}.tar.gz -P /tmp


    Once the Tomcat tar file is downloaded, extract it to the /opt/tomcat directory:

sudo tar -xf /tmp/apache-tomcat-${VERSION}.tar.gz -C /opt/tomcat/

    we’ll create a symbolic link named latest, that will point to the Tomcat installation directory:


sudo ln -s /opt/tomcat/apache-tomcat-${VERSION} /opt/tomcat/latest

    Later, when you need to upgrade your Tomcat instance, simply unpack the newer version and change the symlink to point to it.

    The system user we previously created must have access to the tomcat installation directory. Change the directory ownership to user and group tomcat:

sudo chown -R tomcat: /opt/tomcat

    The shell scripts inside the Tomcat’s bin directory must be executable in order to run:

sudo sh -c 'chmod +x /opt/tomcat/latest/bin/*.sh'

    These scripts are used to start, stop and otherwise manage the Tomcat instance.
    Creating SystemD Unit File

    Instead of directly executing the shell scripts to start and stop the Tomcat server, we’ll run them through a systemd unit file. This way, Tomcat will run as a service.

    Open your text editor and create a tomcat.service unit file in the /etc/systemd/system/ directory:

sudo nano /etc/systemd/system/tomcat.service 
    Paste the following configuration:


/etc/systemd/system/tomcat.service
[Unit]
Description=Tomcat 10 servlet container
After=network.target

[Service]
Type=forking

User=tomcat
Group=tomcat

Environment="JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64"
Environment="JAVA_OPTS=-Djava.security.egd=file:///dev/urandom -Djava.awt.headless=true"

Environment="CATALINA_BASE=/opt/tomcat/latest"
Environment="CATALINA_HOME=/opt/tomcat/latest"
Environment="CATALINA_PID=/opt/tomcat/latest/temp/tomcat.pid"
Environment="CATALINA_OPTS=-Xms512M -Xmx1024M -server -XX:+UseParallelGC"

ExecStart=/opt/tomcat/latest/bin/startup.sh
ExecStop=/opt/tomcat/latest/bin/shutdown.sh

[Install]
WantedBy=multi-user.target


    Modify the JAVA_HOME variable if the path to your Java installation is different.
    Save and close the file and run the following command to notify systemd that we created a new unit file:

sudo systemctl daemon-reload

Enable and start the Tomcat service:


sudo systemctl enable --now tomcat

Check the service status:

sudo systemctl status tomcat

The output should show that the Tomcat server is enabled and running:

● tomcat.service - Tomcat 10 servlet container
     Loaded: loaded (/etc/systemd/system/tomcat.service; enabled; vendor preset: enabled)
     Active: active (running) since Sat 2022-12-24 18:53:37 UTC; 6s ago
    Process: 5124 ExecStart=/opt/tomcat/latest/bin/startup.sh (code=exited, status=0/SUCCESS)
   Main PID: 5131 (java)
...
Copy
You can start, stop, and restart Tomcat the same as any other systemd service:

sudo systemctl start tomcat
sudo systemctl stop tomcat
sudo systemctl restart tomcat

Configuring Firewall
    If you are using a firewall to filter the traffic and want to access Tomcat from outside your local network, you need to open port 8080. Use the following command to open the port:

sudo ufw allow 8080/tcp

Configuring Tomcat Web Management Interface
    At this point, you should be able to access Tomcat using a web browser on port 8080. The web management interface is not accessible because we have not created a user yet.


    Tomcat users and roles are defined in the tomcat-users.xml file. By default, this file includes comments and examples showing you how to create a user or role.

    In this example, we’ll create a user with “admin-gui” and “manager-gui” roles.

    The “admin-gui” role allows the user to access the /host-manager/html URL and create, delete, and otherwise manage virtual hosts. The “manager-gui” role allows the user to deploy and undeploy web applications without restarting the container through the /host-manager/html interface.

    Open the tomcat-users.xml file with your text editor and create a new user, as shown below:

sudo nano /opt/tomcat/latest/conf/tomcat-users.xml  
/opt/tomcat/latest/conf/tomcat-users.xml
<tomcat-users>
<!--
    Comments
-->
   <role rolename="admin-gui"/>
   <role rolename="manager-gui"/>
   <user username="admin" password="admin_password" roles="admin-gui,manager-gui"/>
</tomcat-users>

    Make sure you change the username and password to something more secure.


    By default, the Tomcat web management interface is configured only to allow access to the Manager and Host Manager apps from the localhost. If you want to access the web interface from a remote IP, you must to remove these restrictions. This may have various security implications and is not recommended for production systems.

    To enable access to the web interface from anywhere, open the following two files and comment or remove the lines highlighted in yellow.

    For the Manager app:

sudo nano /opt/tomcat/latest/webapps/manager/META-INF/context.xml 
    For the Host Manager app:

sudo nano /opt/tomcat/latest/webapps/host-manager/META-INF/context.xml

context.xml
<Context antiResourceLocking="false" privileged="true" >
<!--
  <Valve className="org.apache.catalina.valves.RemoteAddrValve"
         allow="127\.\d+\.\d+\.\d+|::1|0:0:0:0:0:0:0:1" />
-->
</Context>

    If you want to access the web interface only from a specific IP, instead of commenting the blocks add your public IP to the list.

    Let’s say your public IP is 41.41.41.41, and you want to allow access only from that IP:

context.xml
<Context antiResourceLocking="false" privileged="true" >
  <Valve className="org.apache.catalina.valves.RemoteAddrValve"
         allow="127\.\d+\.\d+\.\d+|::1|0:0:0:0:0:0:0:1|41.41.41.41" />
</Context>

    The list of allowed IP addresses is a list separated by a vertical bar |. You can add single IP addresses or use regular expressions.

    Once done, restart the Tomcat service for changes to take effect:

sudo systemctl restart tomcat

    Test the Tomcat Installation
    Open your browser and type: http://<your_domain_or_IP_address>:8080

    Assuming the installation is successful, a screen similar to the following should appear:

    Tomcat 8.5
    Tomcat web application manager is available at: http://<your_domain_or_IP_address>:8080/manager/html.

    Tomcat web application manager
    Tomcat virtual host manager is available at: http://<your_domain_or_IP_address>:8080/host-manager/html.

    Tomcat virtual host manager
    Conclusion
    We’ve shown you how to install Tomcat 10.0 on Ubuntu 22.04 and access the Tomcat management interface.

    For more information about Apache Tomcat, visit the official documentation page .

    If you hit a problem or have feedback, leave a comment below.




