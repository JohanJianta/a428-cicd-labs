# a428-cicd-labs

This project sets up a CI/CD pipeline using Jenkins running in a Docker container, complemented by a separate Blue Ocean container for an enhanced UI experience. Within this pipeline, a React application is built and deployed inside a nested Docker container—managed by the Blue Ocean container—demonstrating Docker-in-Docker (DinD) approach to container orchestration.

## Requirements

- Docker 28.0.1
- Git 2.43.0.windows.1
- Visual Studio Code 1.98.2

## Steps

1. Fork & Clone React App Repository
2. Run Jenkins in Docker
3. Prepare Jenkins Wizard
4. Create Pipeline Project in Jenkins
5. Create Jenkins Pipeline with Jenkinsfile

### Fork & Clone React App Repository

1. Make sure you are logged in to GitHub with your own account.
2. Fork [React App repository](https://github.com/dicodingacademy/a428-cicd-labs/tree/react-app) from Dicoding Academy to your GitHub account. Make sure to **not check the "copy the main branch only" option**.
3. Clone the forked React App to your local environment. You can do it from CMD Terminal by using command `git clone -b react-app https://github.com/[YOUR-GITHUB-ACCOUNT]/a428-cicd-labs.git`. Make sure to store it on your preferred directory using `cd` command, for example `cd C:\USERS\ABC\Downloads`.
4. After successfully cloning, open the **a428-cicd-labs** folder with Visual Studio Code. You can do it from CMD Terminal by changing the directory to **a428-cicd-labs** folder first, for example `cd C:\USERS\ABC\Downloads\a428-cicd-labs`, then insert the command `code .` to open it on Visual Studio Code.

### Run Jenkins in Docker

1. Open CMD Terminal on your PC.
2. Insert command `docker network create jenkins` to create bridge network in Docker.
3. Install `docker:dind` Docker image by using this command below.

   ```console
   docker run^
     --name jenkins-docker^
     --detach^
     --privileged^
     --network jenkins^
     --network-alias docker^
     --env DOCKER_TLS_CERTDIR=/certs^
     --volume jenkins-docker-certs:/certs/client^
     --volume jenkins-data:/var/jenkins_home^
     --publish 2376:2376^
     --publish 3000:3000^
     --restart always^
     docker:dind^
     --storage-driver overlay2
   ```

4. Open the current directory on Visual Studio Code by inserting command `code .` on CMD Terminal. On the **Explorer** panel, click the **New File** icon. Name the file as **Dockerfile** without any extension.
5. Paste these commands below inside the newly created Dockerfile.

   ```js
   FROM jenkins/jenkins:2.426.3-jdk11
   USER root
   RUN apt-get update && apt-get install -y lsb-release
   RUN curl -fsSLo /usr/share/keyrings/docker-archive-keyring.asc https://download.docker.com/linux/debian/gpg
   RUN echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.asc] https://download.docker.com/linux/debian $(lsb_release -cs) stable" > /etc/apt/sources.list.d/docker.list
   RUN apt-get update && apt-get install -y docker-ce-cli
   USER jenkins
   RUN jenkins-plugin-cli --plugins "blueocean:1.25.5 docker-workflow:1.28"
   ```

6. After saving the Dockerfile, back to previous CMD Terminal. Make sure the current directory is the same as where the Dockerfile is. Next, create a new Docker image using command `docker build -t myjenkins-blueocean:2.426.3-1 .`
7. Run the **myjenkins-blueocean:2.426.3-1** Docker image as container by using this command below. Please take a note at `[YOUR-REPOSITORY-PATH]`. Make sure to change it to your **a428-cicd-labs** folder path from previous step, for example `C:\USERS\ABC\Downloads\a428-cicd-labs`. This is to make sure your **a428-cicd-labs** folder is mapped correctly to **/home/a428-cicd-labs** directory on container, which would be used for later step.

   ```console
   docker run^
     --name jenkins-blueocean^
     --detach^
     --network jenkins^
     --env DOCKER_HOST=tcp://docker:2376^
     --env DOCKER_CERT_PATH=/certs/client^
     --env DOCKER_TLS_VERIFY=1^
     --publish 8080:8080^
     --publish 50000:50000^
     --volume jenkins-data:/var/jenkins_home^
     --volume jenkins-docker-certs:/certs/client:ro^
     --volume "%USERPROFILE%:/home"^
     --volume "[YOUR-REPOSITORY-PATH]:/home/a428-cicd-labs"^
     --restart=on-failure^
     --env JAVA_OPTS="-Dhudson.plugins.git.GitSCM.ALLOW_LOCAL_CHECKOUT=true"^
     myjenkins-blueocean:2.426.3-1
   ```

8. The Jenkins is now running on Docker.

### Prepare Jenkins Wizard

1. Open your browser and go to `http://localhost:8080`. Wait until _Unlock Jenkins_ page show up.
2. As written on the page, you need to paste the password from **Jenkins log**. You can access it from CMD Terminal by using command `docker logs jenkins-blueocean`. The password is located in between two asterisk sequence. After pasting the password to **Administrator password** column, click **Continue**.
3. On _Customize Jenkins_ page, select **Install suggested plugins** and wait for the setup wizard to finish.
4. After the setup finished, you need to create an administrator user. On _Create First Admin User_ page, fill up the form and click **Save and Continue**.
5. On _Instance Configuration_ page, click **Save and Finish**. When _Jenkins is ready_ page show up, click **Start using Jenkins**. If the page shows _Jenkins is almost ready!_ instead, you can click **Restart** or refresh the page manually.
6. After that, you can log in to Jenkins using the previously created credential.

### Create Pipeline Project in Jenkins

1. Open Jenkins page, or go to `http://localhost:8080` and log in again with your credential.
2. Select **Create a job**, or click **New Item** on top left.
3. On _Enter an item name_ column, input the pipeline name as you wish, for example **react-app**, then click **OK**.
4. On the next page, input brief description for your pipeline on Description column, for example **A simple pipeline for React App project**.
5. Scroll down to _Pipeline section_. On _Definition_ column, choose **Pipeline script from SCM** option. Then on **SCM** column, select **Git**.
6. On _Repository URL_ column, input the **a428-cicd-labs** folder that has previously mapped on container, which is **/home/a428-cicd-labs**.
7. Lastly, scroll down to _Branch Specifier_ column and change it to **\*/react-app**, then click **Save**.

### Create Jenkins Pipeline with Jenkinsfile

1. Go to previously opened Visual Studio Code for **a428-cicd-labs** folder.
2. On the **Explorer** panel, click the **New File** icon and name it as **Jenkinsfile** without any extension.
3. Paste this Declarative Pipeline code below inside the **Jenkinsfile** and save it.

   ```ruby
   pipeline {
       agent {
           docker {
               image 'node:16-buster-slim'
               args '-p 3000:3000'
           }
       }
       stages {
           stage('Build') {
               steps {
                   sh 'npm install'
               }
           }
           stage('Test') {
               steps {
                   sh './jenkins/scripts/test.sh'
               }
           }
       }
   }
   ```

4. Commit the file to local repository by using this command below.

   ```console
   git add .
   git commit -m "Add initial Jenkinsfile"
   ```

5. Go back to Jenkins page, and now click **Open Blue Ocean** on the left side.
6. Select **react-app**, then click **Run**. After that, quickly click on the **OPEN** link that show up at bottom-right corner. There you can see how Jenkins runs your Pipeline project from _Build_ to Test steps. This process may take a couple minutes for the first time.
7. If the Pipeline process success, the Blue Ocean interface will turn to green. On the other hand, the interface would turn to red instead if the process failed.
8. Click on the cross mark (X) on top-right corner to go back to the Blue Ocean main page.
9. The Pipeline project is complete, you can now stop the Docker container by using command `docker stop jenkins-blueocean jenkins-docker` on CMD Terminal.
