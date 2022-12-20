
# Sonarqube Setup Guide

## 🧰 Prerequisites
1. Create an EC2 instance (min t2.small),
2. Make sure port 9000(or port on which SonarQube is running) is opened at security group

3. Install Java-11
  ```sh 
   apt-get update   
   apt  list | grep openjdk-11  
   apt-get install openjdk-11-jdk -y   
   ```

## Install & Setup Postgres Database for SonarQube
`Source: https://www.postgresql.org/download/linux/ubuntu/`  
1. Install Postgres database   
  ```sh 
  sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'  
  wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -
  sudo apt-get update
  sudo apt-get -y install postgresql
  ```

2. Set a password and connect to database (setting password as "admin" password)
  ```sh 
  sudo passwd postgres
  su - postgres
  ```

3. Create a database user and database for sonarque 
  ```sh 
  createuser sonar
  psql
  ALTER USER sonar WITH ENCRYPTED PASSWORD 'admin';
  CREATE DATABASE sonarqube OWNER sonar;
  GRANT ALL PRIVILEGES ON DATABASE sonarqube to sonar;
  ``` 

4. Restart postgres database to take latest changes effect 
  ```sh 
  systemctl restart postgresql 
  systemctl status postgresql
  ```
`check point`: You should see postgres is running on 5432
```
apt install net-tools
netstat -tulpn
```
<hr/>

## Update configuration files
`Source: https://docs.sonarqube.org/latest/requirements/requirements/`

1. Added below entries in `/etc/sysctl.conf`
  ```sh 
  vm.max_map_count=524288
  fs.file-max=131072
  ulimit -n 131072
  ulimit -u 8192
  ```
2. Add below entries in `/etc/security/limits.conf`
  ```sh 
  sonarqube   -   nofile   131072
  sonarqube   -   nproc    8192
  ```

3. reboot the server 
  ```sh 
  init 6
  ```
  ==========================================

 ## SonarQube Setup

1. Download [soarnqube](https://www.sonarqube.org/downloads/) and extract it.   
  ```sh 
  wget https://binaries.sonarsource.com/Distribution/sonarqube/sonarqube-8.9.2.46101.zip
  unzip sonarqube-8.9.2.46101.zip
  ```

2. Update sonar.properties with below information 
  ```sh
  sonar.jdbc.username=<sonar_database_username>
  sonar.jdbc.password=<sonar_database_password>

  #sonar.jdbc.username=sonar
  #sonar.jdbc.password=admin
  #sonar.jdbc.url=jdbc:postgresql://localhost/sonarqube
  #sonar.search.javaOpts=-Xmx512m -Xms512m -XX:MaxDirectMemorySize=256m -XX:+HeapDumpOnOutOfMemoryError
  ```

3. Create a `/etc/systemd/system/sonarqube.service` file start sonarqube service at the boot time 
  ```sh   
  cat >> /etc/systemd/system/sonarqube.service <<EOL
  [Unit]
  Description=SonarQube service
  After=syslog.target network.target

  [Service]
  Type=forking
  User=sonar
  Group=sonar
  PermissionsStartOnly=true
  ExecStart=/opt/sonarqube/bin/linux-x86-64/sonar.sh start 
  ExecStop=/opt/sonarqube/bin/linux-x86-64/sonar.sh stop
  StandardOutput=syslog
  LimitNOFILE=65536
  LimitNPROC=4096
  TimeoutStartSec=5
  Restart=always

  [Install]
  WantedBy=multi-user.target
  EOL
  ```

4. Add sonar user and grant ownership to /opt/sonarqube directory 
  ```sh 
  useradd -d /opt/sonarqube sonar
  chown -R sonar:sonar
  ```

5. Reload the demon and start sonarqube service 
  ```sh 
  systemctl daemon-reload 
  systemctl start sonarqube.service 
  ```

## Setup GitHub Actions for Sonar scanner

- Create your GitHub Secrets.
- Configure your workflow YAML file.
- Commit and push your code to start the analysis.

### Creating your GitHub secrets
You can create repository secrets from your GitHub repository:
- On GitHub.com, navigate to the main page of the repository.
- Under your repository name, click  Settings.
- Repository settings button
- In the "Security" section of the sidebar, select  Secrets, then click Actions.
- Click New repository secret.
- Type a name for your secret in the Name input box.
- Enter the value for your secret.
- Click Add secret.

You need to set the following GitHub repository secrets to analyze your projects with GitHub Actions:

SONARQUBE_HOST - AWS instance address <br>
SONARQUBE_TOKEN - generate in SonarQube <br>
SONARQUBE_PROJECT - project name to use in SonarQube <br>

### Configuring your .github/workflows/build.yml file

1. Add `.github/workflows/build.yml` directory in repository.
2. Write the following in your workflow YAML file:
```
on:
  push:
    branches:
      - main # or the name of your main branch
  pull_request:
    types: [opened, synchronize, reopened]

name: SonarQube
jobs:
  sonarqube:
    name: SonarQube Build
    runs-on: ubuntu-latest
    steps:
      - name: Checking out
        uses: actions/checkout@main
        with:
          fetch-depth: 0  # Shallow clones should be disabled for a better relevancy of analysis
      - name: Cache SonarQube packages
        uses: actions/cache@v3
        with:
          path: ~/ #path to packages cache
          key: ${{ runner.os }}-sonar
          restore-keys: ${{ runner.os }}-sonar
      - name: Cache SonarQube scanner
        id: cache-sonar-scanner
        uses: actions/cache@v3
        with:
          path: ~/ #path to scanner cache
          key: ${{ runner.os }}-sonar-scanner
          restore-keys: ${{ runner.os }}-sonar-scanner
      - name: Install SonarQube scanner
        if: steps.cache-sonar-scanner.outputs.cache-hit != 'true'
        run: dotnet tool install --global dotnet-sonarscanner
      - name: Build and scan
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}  # Needed to get PR information, if any
        run: |
          dotnet sonarscanner begin /k:${{ secrets.SONARQUBE_PROJECT }} /d:sonar.host.url=${{ secrets.SONARQUBE_HOST }}  /d:sonar.login=${{ secrets.SONARQUBE_TOKEN }}
          dotnet build
          dotnet sonarscanner end /d:sonar.login=${{ secrets.SONARQUBE_TOKEN }}
```
## Commit and push your code
Commit and push your code to start the analysis. Each new push you make on your branches or pull requests will trigger a new analysis in SonarQube.
