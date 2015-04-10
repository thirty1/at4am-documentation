# [![AT4AM.eu](https://at4am.eu/resource/image/logo/at4ameu-32x32.jpg)](https://at4am.eu/) Running the [e-parliament AT4AM (NSESA) fork](https://github.com/e-parliament)

Steps for fetching/compiling/installing/configuring/running/testing the [e-parliament fork of AT4AM](https://github.com/e-parliament). Let me know if they work for you!


## Preparations

I'm sure these software versions have changed since I first installed them, so adapt your usage -- and please update this document.


**Assumptions**

- You have 5-10 GB of disk space, depending on if you already have all software and Maven dependencies installed or not. (GWT's SOYC reports take up a lot of space; afaik they aren't useful at all for this project and the `soycReport` directories can be deleted.)
- You make sure the environment variable `AT4AM_TOMCAT` used in the script below is set properly.
- Any previously installed `~/nsesa-server.properties`, `editor.war` and `services.war` can and will be overwritten.
- You will search-replace the instructions' `<database-password>` with a password generated by `openssl rand -base64 24`.
- You will search-replace the instructions' `<tomcat-password>` with a password you choose. (If you don't change it, Tomcat will throw a configuration error.)


**My setup**

Software was installed using [Homebrew](http://brew.sh/) and [Homebrew-Cask](http://caskroom.io/).

- Mac OS X 10.10 Yosemite
- Tomcat 7: `brew install homebrew/versions/tomcat7`
- MySQL 5.6: `brew install mysql`
- Java 1.7.0_45: `brew cask install java`
- Maven 3.2.5: `brew install maven`


## Steps

### Fetch, compile, install

```bash
# This path to Tomcat might differ on your system.
# For me it expands to `/usr/local/opt/tomcat7`.
AT4AM_TOMCAT=$(brew --prefix tomcat7)

# Had to explicitly set JAVA_HOME before compiling. You might have a different version.
export JAVA_HOME="$(/usr/libexec/java_home -v 1.7)"

# Creating two directory levels, as logs will be created one level above the execution folder.
mkdir at4am
cd at4am
mkdir e-parliament
cd e-parliament

nsesaProjects=(editor editor-an server-impl server-api standalone diff)

# Clone code from github.
for project in "${nsesaProjects[@]}";
do
	# Optionally use gitslave (gits attach).
	git clone "https://github.com/e-parliament/nsesa-$project.git" "nsesa-$project"
done

git clone https://github.com/e-parliament/e-parliament.github.com.git e-parliament.github.com

# Compile projects.
for project in "${nsesaProjects[@]}";
do
	pushd "nsesa-$project" >/dev/null

	# This will take quite some time (maybe 10-30 minutes) to complete for some projects.
	mvn install

	popd >/dev/null
done

# Remove any old AT4AM instance.
rm -r "$AT4AM_TOMCAT"/libexec/webapps/{editor,services}/

# Copy newly compiled code to Tomcat.
cp ./nsesa-editor-an/target/editor.war ./nsesa-server-impl/target/services.war "$AT4AM_TOMCAT/libexec/webapps/"
```


### Configure

Set up AT4AM to use MySQL, then access your MySQL server to create a user and database:

```bash
# Configure AT4AM to use MySQL.
read -d '' mysqlSettings <<-'EOF'
jpa.database=MYSQL
jdbc.driver=com.mysql.jdbc.Driver
jdbc.url=jdbc:mysql://localhost:3306/nsesa
jdbc.username=nsesa
jdbc.password=<database-password>
EOF

echo "$mysqlSettings" >~/nsesa-server.properties


# MySQL might already be running on your system.
# mysql.server start
mysql -u root
```

Execute these lines:

```sql
CREATE USER 'nsesa'@'localhost' IDENTIFIED BY '<database-password>';
CREATE DATABASE nsesa;
GRANT ALL PRIVILEGES ON nsesa.* TO 'nsesa'@'localhost';
```


Add a user in Tomcat's authentication system:

```bash
# Open up tomcat-users.xml for editing.
"$EDITOR" "$AT4AM_TOMCAT/libexec/conf/tomcat-users.xml"
```

```xml
<!-- Add these three lines inside of <tomcat-users> -->
  <role rolename="ROLE_ADMIN"/>
  <role rolename="ROLE_USER"/>
  <user username="nsesa" password="<tomcat-password>" roles="ROLE_ADMIN,ROLE_USER"/>
```



### Run

```bash
# Start Tomcat in the current terminal window.
# If Tomcat was already running, you might want to find a way to look at the console output.
# Exit with Ctrl+c when you're tired of looking at it.
catalina run

# Peek at logs.
tail ../logs/*.log
```



## Test

- Make sure the editor and services are deployed and running: [http://localhost:8080/manager/html/](http://localhost:8080/manager/html/)
- See if you can load the login page: [http://localhost:8080/editor/](http://localhost:8080/editor/)
- Log in with the credentials entered in `tomcat-users.xml`.
- Try loading the first example document to amend, draft or look at the XML:
  - [http://localhost:8080/editor/amendment.html?documentID=1](http://localhost:8080/editor/amendment.html?documentID=1)
  - [http://localhost:8080/editor/drafter.html?documentID=1](http://localhost:8080/editor/drafter.html?documentID=1)
  - [http://localhost:8080/editor/markup.html?documentID=1](http://localhost:8080/editor/markup.html?documentID=1)


---

[![AT4AM.eu](https://at4am.eu/resource/image/logo/at4ameu-16x16.jpg)](https://at4am.eu/) [AT4AM.eu](https://at4am.eu/) &copy; 2013, 2014, 2015 [Föreningen för digitala fri- och rättigheter (DFRI)](https://dfri.se/). The documentation is released under the [Creative Commons Attribution-ShareAlike 4.0 International (CC BY-SA 4.0)](https://creativecommons.org/licenses/by-sa/4.0/) license. Related resources and projects may have other licences.