# Primera parte

Create a bridge network in Docke
```sh
docker network create jenkins
```

![alt text](image.png)

In order to execute Docker commands inside Jenkins nodes, download and run the docker:dind Docker image

```sh
docker run \
  --name jenkins-docker \
  --rm \
  --detach \
  --privileged \
  --network jenkins \
  --network-alias docker \
  --env DOCKER_TLS_CERTDIR=/certs \
  --volume jenkins-docker-certs:/certs/client \
  --volume jenkins-data:/var/jenkins_home \
  --publish 2376:2376 \
  docker:dind \
  --storage-driver overlay2
```
![alt text](image-1.png)

Customize the official Jenkins Docker image

```sh
cd ..
nano Dockerfile
```

```Dockerfile
    FROM jenkins/jenkins:2.492.1-jdk17
    USER root
    RUN apt-get update && apt-get install -y lsb-release
    RUN curl -fsSLo /usr/share/keyrings/docker-archive-keyring.asc \
    https://download.docker.com/linux/debian/gpg
    RUN echo "deb [arch=$(dpkg --print-architecture) \
    signed-by=/usr/share/keyrings/docker-archive-keyring.asc] \
    https://download.docker.com/linux/debian \
    $(lsb_release -cs) stable" > /etc/apt/sources.list.d/docker.list
    RUN apt-get update && apt-get install -y docker-ce-cli
    USER jenkins
    RUN jenkins-plugin-cli --plugins "blueocean docker-workflow"
```

Build a new docker image from this Dockerfile, and assign the image a meaningful name, such as "myjenkins-blueocean:2.492.1-1":

```sh
docker build -t myjenkins-blueocean:2.492.1-1 .
```

![alt text](image-2.png)

Run your own myjenkins-blueocean:2.492.1-1 image as a container in Docker

```sh
  docker run \
    --name jenkins-blueocean \
    --restart=on-failure \
    --detach \
    --network jenkins \
    --env DOCKER_HOST=tcp://docker:2376 \
    --env DOCKER_CERT_PATH=/certs/client \
    --env DOCKER_TLS_VERIFY=1 \
    --publish 8080:8080 \
    --publish 50000:50000 \
    --volume jenkins-data:/var/jenkins_home \
    --volume jenkins-docker-certs:/certs/client:ro \
    myjenkins-blueocean:2.492.1-1
```

![alt text](image-3.png)


![alt text](image-5.png)
```sh
docker logs jenkins-blueocean
```

```sh
752b699f61024c06aad14779353911dc
```

On the Unlock Jenkins page, paste this password into the Administrator password field and click Continue.

![alt text](image-4.png)


After unlocking Jenkins, the Customize Jenkins page appears. Install suggested plugins

![alt text](image-6.png)

Skip

![alt text](image-7.png)

When the Jenkins is ready page appears, click Start using Jenkins.
![alt text](image-8.png)

# Segunda parte

Instalar un agente de Jenkins y conectarlo con el controlador de Jenkins (El agente y el controlador deben estar en la misma red).

In a terminal window

```sh
  ssh-keygen -f ~/.ssh/jenkins_agent_key
```

![alt text](image-10.png)

Go to your Jenkins dashboard

![alt text](image-11.png)

Go to ```Manage Jenkins``` option in main menu and click on the Manage Credentials button

![alt text](image-13.png)

Select the drop option Add Credentials from the global item

![alt text](image-14.png)

Fill in the form:

Kind: SSH Username with private key;

id: jenkins

description: The jenkins ssh key

username: jenkins

Private Key: select ```Enter directly``` and press the Add button to insert the content of your private key file at ```~/.ssh/jenkins_agent_key```

Passphrase: fill your passphrase used to generate the SSH key pair (leave empty if you didn’t use one at the previous step) and then press the ```Create``` button

![alt text](image-15.png)

Here we will use the ```docker-ssh-agent``` image to create the agent containers.

```sh
cat ~/.ssh/jenkins_agent_key.pub

docker run -d --rm --name=agent1 --network jenkins -p 4444:22 -e "JENKINS_AGENT_SSH_PUBKEY=ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQCv0lBlJ+6PMrgw5yQpsF4a/izRBOj2CQfb3jtrCNPeByVkZdHLxH5sE1y7dTQ+ZAvySy9b5ut1fC2S0HfuLlCCrVWhSB8YjRQNdIMKzx+p0FfjdR1GxJBGNqsygptuN8Kq5vfj9LX9InKt6wf//FStd0ECcga6qbC+yxekD3x1sRfWs3/sCTuzBuPLmplgH6HgwcwsdF4vlmx4bq8ejJpTjE8wmBmPsUSapKKxttX2HX+PYhSm3p4ldWzuYEu8dd98PEad9zLv9cSf05ulTOw1W86KYaSurZK8GWo1uAo8+eb5p/n/lQ+U/DBVTo3VSQlpFywbQhIsV2Ejh5RLluwxcvDFYTObZDClDS4PiyhUUsoRCPWbEpyFzsfshMOYsY3ZF5x7DY62GVUR6HbwhUBCJ1Kh1X9irYhiJmWNBlE99fUdpF0MMPyANcssCbR2yUBXrB2h440dpKTkFVrTmi7K+NSIolC7m9C8jQBJRwuD+/Iibq2VQL3p3vQ5THrvkYk=" jenkins/ssh-agent:alpine-jdk17
```

![alt text](image-17.png)

Verify the agent

```sh
docker logs agent1
```
![alt text](image-18.png)

Ingresar a la configuración de Jenkins, Nodos

![alt text](image-19.png)

Crear un nuevo nodo que se lance al conectarlo al controlador.

![alt text](image-20.png)

![alt text](image-21.png)

Correr el comando generado por Jenkins para iniciar y conectar el agente.

```sh
mkdir agent
```

Seleccionar el nodo

![alt text](image-22.png)

Relanzar agente

![alt text](image-23.png)


Si no funciona, podemos intentar pasar la clave directamente a jenkins

```sh
docker exec -it jenkins-blueocean sh -c "echo '-----BEGIN OPENSSH PRIVATE KEY----- ...' > /var/jenkins_home/.ssh/id_rsa"
docker exec -it jenkins-blueocean sh -c "chmod 600 /var/jenkins_home/.ssh/id_rsa"
```

![alt text](image-24.png)

# Tercera parte

Crear un ítem de tipo pipeline en Jenkins y configurar el script usando una aplicación sencilla (e.j., https://github.com/jenkins-docs/simple-java-maven-app). Ejecutar el pipeline y revisar la consola.

![alt text](image-25.png)

Crear tarea

![alt text](image-27.png)

En la página de configuración del pipeline, desplázate hacia abajo hasta la sección "Pipeline".

En la opción "Definition", selecciona "Pipeline script from SCM".

![alt text](image-28.png)

En "SCM", selecciona "Git".

En "Repository URL", ingresa la URL del repositorio de ejemplo:

Cambiar el path del Jenkinsfile si no esta en la raiz

![alt text](image-29.png)

Una vez guardado, serás redirigido a la página del pipeline.

Haz clic en "Construir ahora" (o "Build Now") para ejecutar el pipeline.

![alt text](image-30.png)

Mirar la consola

![alt text](image-31.png)

Da error porque el maven que esta jalando el jenkins no es el que pide en el pom.xml

Crear un ítem de tipo pipeline en Jenkins y configurar el script usando Github+Maven. Ejecutar el pipeline y revise la consola.

En la página de configuración del pipeline, desplázate hacia abajo hasta la sección "Pipeline".

En la opción "Definition", selecciona "Pipeline script".

Seleccionar Github+Maven

![alt text](image-32.png)

Ejecutar el pipeline y revise la consola.

```
Started by user admin

Obtained jenkins/Jenkinsfile from git https://github.com/jenkins-docs/simple-java-maven-app
[Pipeline] Start of Pipeline
[Pipeline] node
Running on Jenkins
 in /var/jenkins_home/workspace/simple-java-maven-app
[Pipeline] {
[Pipeline] stage
[Pipeline] { (Declarative: Checkout SCM)
[Pipeline] checkout
Selected Git installation does not exist. Using Default
The recommended git tool is: NONE
No credentials specified
 > git rev-parse --resolve-git-dir /var/jenkins_home/workspace/simple-java-maven-app/.git # timeout=10
Fetching changes from the remote Git repository
 > git config remote.origin.url https://github.com/jenkins-docs/simple-java-maven-app # timeout=10
Fetching upstream changes from https://github.com/jenkins-docs/simple-java-maven-app
 > git --version # timeout=10
 > git --version # 'git version 2.39.5'
 > git fetch --tags --force --progress -- https://github.com/jenkins-docs/simple-java-maven-app +refs/heads/*:refs/remotes/origin/* # timeout=10
 > git rev-parse refs/remotes/origin/master^{commit} # timeout=10
Checking out Revision 05f17fbceabf683500da604bba18c6b8d7e3f109 (refs/remotes/origin/master)
 > git config core.sparsecheckout # timeout=10
 > git checkout -f 05f17fbceabf683500da604bba18c6b8d7e3f109 # timeout=10
Commit message: "Merge pull request #1090 from jenkins-docs/dependabot/maven/org.apache.maven.plugins-maven-compiler-plugin-3.14.0"
First time build. Skipping changelog.
[Pipeline] }
[Pipeline] // stage
[Pipeline] withEnv
[Pipeline] {
[Pipeline] isUnix
[Pipeline] withEnv
[Pipeline] {
[Pipeline] sh
+ docker inspect -f . maven:3.9.0
.
[Pipeline] }
[Pipeline] // withEnv
[Pipeline] withDockerContainer
Jenkins seems to be running inside container 536f8b7b9379fa4822662cd3365920932964ae59eebb82511539a529eb38485a
but /var/jenkins_home/workspace/simple-java-maven-app could not be found among []
but /var/jenkins_home/workspace/simple-java-maven-app@tmp could not be found among []
$ docker run -t -d -u 1000:1000 -v /root/.m2:/root/.m2 -w /var/jenkins_home/workspace/simple-java-maven-app -v /var/jenkins_home/workspace/simple-java-maven-app:/var/jenkins_home/workspace/simple-java-maven-app:rw,z -v /var/jenkins_home/workspace/simple-java-maven-app@tmp:/var/jenkins_home/workspace/simple-java-maven-app@tmp:rw,z -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** -e ******** maven:3.9.0 cat
$ docker top 0ab1fd23720ae0b95b27b5cdb4508a7cea875bcda0ee671f3f1bdb9fae52b7e6 -eo pid,comm
[Pipeline] {
[Pipeline] stage
[Pipeline] { (Build)
[Pipeline] sh
+ mvn -B -DskipTests clean package
[INFO] Scanning for projects...
[INFO] 
[INFO] ----------------------< com.mycompany.app:my-app >----------------------
[INFO] Building my-app 1.0-SNAPSHOT
[INFO]   from pom.xml
[INFO] --------------------------------[ jar ]---------------------------------
[INFO] 
[INFO] --- clean:3.2.0:clean (default-clean) @ my-app ---
[INFO] 
[INFO] --- enforcer:3.5.0:enforce (enforce-maven) @ my-app ---
[INFO] Rule 1: org.apache.maven.enforcer.rules.version.RequireJavaVersion passed
[INFO] ------------------------------------------------------------------------
[INFO] BUILD FAILURE
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  0.548 s
[INFO] Finished at: 2025-03-01T23:25:20Z
[INFO] ------------------------------------------------------------------------
[ERROR] Failed to execute goal org.apache.maven.plugins:maven-enforcer-plugin:3.5.0:enforce (enforce-maven) on project my-app: 
[ERROR] Rule 0: org.apache.maven.enforcer.rules.version.RequireMavenVersion failed with message:
[ERROR] Detected Maven Version: 3.9.0 is not in the allowed range [3.9.2,).
[ERROR] -> [Help 1]
[ERROR] 
[ERROR] To see the full stack trace of the errors, re-run Maven with the -e switch.
[ERROR] Re-run Maven using the -X switch to enable full debug logging.
[ERROR] 
[ERROR] For more information about the errors and possible solutions, please read the following articles:
[ERROR] [Help 1] http://cwiki.apache.org/confluence/display/MAVEN/MojoExecutionException
[Pipeline] }
[Pipeline] // stage
[Pipeline] stage
[Pipeline] { (Test)
Stage "Test" skipped due to earlier failure(s)
[Pipeline] getContext
[Pipeline] }
[Pipeline] // stage
[Pipeline] stage
[Pipeline] { (Deliver)
Stage "Deliver" skipped due to earlier failure(s)
[Pipeline] getContext
[Pipeline] }
[Pipeline] // stage
[Pipeline] }
$ docker stop --time=1 0ab1fd23720ae0b95b27b5cdb4508a7cea875bcda0ee671f3f1bdb9fae52b7e6
$ docker rm -f --volumes 0ab1fd23720ae0b95b27b5cdb4508a7cea875bcda0ee671f3f1bdb9fae52b7e6
[Pipeline] // withDockerContainer
[Pipeline] }
[Pipeline] // withEnv
[Pipeline] }
[Pipeline] // node
[Pipeline] End of Pipeline
ERROR: script returned exit code 1
Finished: FAILURE
```


![alt text](image-33.png)

Modificar el script anterior para que verifique si el proyecto ya existe, en este caso elimine la carpeta y luego lo descargue de nuevo el proyecto. 
```groovy
// agregar estas stages al pipeline
stage('Check and Clean Workspace') {
            steps {
                script {
                    // Define the directory of the project
                    def projectDir = "simple-maven-project-with-tests"

                    // Check if the directory exists
                    if (fileExists(projectDir)) {
                        echo "Project directory exists. Deleting..."
                        // Delete the directory
                        sh "rm -rf ${projectDir}"
                    } else {
                        echo "Project directory does not exist. Proceeding to clone..."
                    }
                }
            }
        }

        stage('Clone Repository') {
            steps {
                // Get some code from a GitHub repository
                git 'https://github.com/jglick/simple-maven-project-with-tests.git'
            }
        }
```
Ejecutar el pipeline y revise la consola.

![alt text](image-34.png)

```
Started by user admin

[Pipeline] Start of Pipeline
[Pipeline] node
Running on Jenkins
 in /var/jenkins_home/workspace/simple-java-maven-app2
[Pipeline] {
[Pipeline] stage
[Pipeline] { (Declarative: Tool Install)
[Pipeline] tool
[Pipeline] envVarsForTool
[Pipeline] }
[Pipeline] // stage
[Pipeline] withEnv
[Pipeline] {
[Pipeline] stage
[Pipeline] { (Check and Clean Workspace)
[Pipeline] tool
[Pipeline] envVarsForTool
[Pipeline] withEnv
[Pipeline] {
[Pipeline] script
[Pipeline] {
[Pipeline] fileExists
[Pipeline] echo
Project directory does not exist. Proceeding to clone...
[Pipeline] }
[Pipeline] // script
[Pipeline] }
[Pipeline] // withEnv
[Pipeline] }
[Pipeline] // stage
[Pipeline] stage
[Pipeline] { (Clone Repository)
[Pipeline] tool
[Pipeline] envVarsForTool
[Pipeline] withEnv
[Pipeline] {
[Pipeline] git
Selected Git installation does not exist. Using Default
The recommended git tool is: NONE
No credentials specified
 > git rev-parse --resolve-git-dir /var/jenkins_home/workspace/simple-java-maven-app2/.git # timeout=10
Fetching changes from the remote Git repository
 > git config remote.origin.url https://github.com/jglick/simple-maven-project-with-tests.git # timeout=10
Fetching upstream changes from https://github.com/jglick/simple-maven-project-with-tests.git
 > git --version # timeout=10
 > git --version # 'git version 2.39.5'
 > git fetch --tags --force --progress -- https://github.com/jglick/simple-maven-project-with-tests.git +refs/heads/*:refs/remotes/origin/* # timeout=10
 > git rev-parse refs/remotes/origin/master^{commit} # timeout=10
Checking out Revision 9a5e9832814aa17fa7e514e5890fc6cf913d9a77 (refs/remotes/origin/master)
 > git config core.sparsecheckout # timeout=10
 > git checkout -f 9a5e9832814aa17fa7e514e5890fc6cf913d9a77 # timeout=10
 > git branch -a -v --no-abbrev # timeout=10
 > git branch -D master # timeout=10
 > git checkout -b master 9a5e9832814aa17fa7e514e5890fc6cf913d9a77 # timeout=10
Commit message: "Creating a main class so there is something for `javadoc:javadoc` to do"
 > git rev-list --no-walk 9a5e9832814aa17fa7e514e5890fc6cf913d9a77 # timeout=10
[Pipeline] }
[Pipeline] // withEnv
[Pipeline] }
[Pipeline] // stage
[Pipeline] stage
[Pipeline] { (Build)
[Pipeline] tool
[Pipeline] envVarsForTool
[Pipeline] withEnv
[Pipeline] {
[Pipeline] sh
+ mvn -Dmaven.test.failure.ignore=true clean package
[INFO] Scanning for projects...
[INFO] 
[INFO] ----------------< test:simple-maven-project-with-tests >----------------
[INFO] Building simple-maven-project-with-tests 1.0-SNAPSHOT
[INFO]   from pom.xml
[INFO] --------------------------------[ jar ]---------------------------------
[INFO] 
[INFO] --- clean:3.2.0:clean (default-clean) @ simple-maven-project-with-tests ---
[INFO] Deleting /var/jenkins_home/workspace/simple-java-maven-app2/target
[INFO] 
[INFO] --- resources:3.3.1:resources (default-resources) @ simple-maven-project-with-tests ---
[INFO] skip non existing resourceDirectory /var/jenkins_home/workspace/simple-java-maven-app2/src/main/resources
[INFO] 
[INFO] --- compiler:3.13.0:compile (default-compile) @ simple-maven-project-with-tests ---
[INFO] Recompiling the module because of changed source code.
[INFO] Compiling 1 source file with javac [debug target 1.8] to target/classes
[WARNING] bootstrap class path not set in conjunction with -source 8
[INFO] 
[INFO] --- resources:3.3.1:testResources (default-testResources) @ simple-maven-project-with-tests ---
[INFO] skip non existing resourceDirectory /var/jenkins_home/workspace/simple-java-maven-app2/src/test/resources
[INFO] 
[INFO] --- compiler:3.13.0:testCompile (default-testCompile) @ simple-maven-project-with-tests ---
[INFO] Recompiling the module because of changed dependency.
[INFO] Compiling 3 source files with javac [debug target 1.8] to target/test-classes
[WARNING] bootstrap class path not set in conjunction with -source 8
[INFO] /var/jenkins_home/workspace/simple-java-maven-app2/src/test/java/test/Base.java: /var/jenkins_home/workspace/simple-java-maven-app2/src/test/java/test/Base.java uses or overrides a deprecated API.
[INFO] /var/jenkins_home/workspace/simple-java-maven-app2/src/test/java/test/Base.java: Recompile with -Xlint:deprecation for details.
[INFO] 
[INFO] --- surefire:2.18.1:test (default-test) @ simple-maven-project-with-tests ---
[INFO] Surefire report directory: /var/jenkins_home/workspace/simple-java-maven-app2/target/surefire-reports

-------------------------------------------------------
 T E S T S
-------------------------------------------------------
Running test.SomeTest
Tests run: 6, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.003 sec - in test.SomeTest
Running test.OtherTest
Tests run: 1, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0 sec - in test.OtherTest

Results :

Tests run: 7, Failures: 0, Errors: 0, Skipped: 0

[INFO] 
[INFO] --- jar:3.4.1:jar (default-jar) @ simple-maven-project-with-tests ---
[INFO] Building jar: /var/jenkins_home/workspace/simple-java-maven-app2/target/simple-maven-project-with-tests-1.0-SNAPSHOT.jar
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  1.627 s
[INFO] Finished at: 2025-03-01T23:43:46Z
[INFO] ------------------------------------------------------------------------
Post stage
[Pipeline] junit
Recording test results
[Checks API] No suitable checks publisher found.
[Pipeline] archiveArtifacts
Archiving artifacts
[Pipeline] }
[Pipeline] // withEnv
[Pipeline] }
[Pipeline] // stage
[Pipeline] }
[Pipeline] // withEnv
[Pipeline] }
[Pipeline] // node
[Pipeline] End of Pipeline
Finished: SUCCESS
```

Agregue un parámetro a Jenkins que permita escoger cual es la rama que se va a realizar la compilación, puede utilizar la rama declarative o testng disponibles en el proyecto de ejemplo

```groovy
    parameters {
        choice(
            name: 'BRANCH',
            choices: ['declarative', 'testng'],
            description: 'Select the branch to build'
        )
    }
```

y en stages reemplazar
```groovy
        stage('Clone Repository') {
            steps {
                // Get some code from a GitHub repository, using the selected branch
                git branch: "${params.BRANCH}", url: 'https://github.com/jglick/simple-maven-project-with-tests.git'
            }
        }
```

```
Ejecutar el pipeline y revise la consola.

Started by user admin

[Pipeline] Start of Pipeline
[Pipeline] node
Running on Jenkins
 in /var/jenkins_home/workspace/simple-java-maven-app2
[Pipeline] {
[Pipeline] stage
[Pipeline] { (Declarative: Tool Install)
[Pipeline] tool
[Pipeline] envVarsForTool
[Pipeline] }
[Pipeline] // stage
[Pipeline] withEnv
[Pipeline] {
[Pipeline] stage
[Pipeline] { (Check and Clean Workspace)
[Pipeline] tool
[Pipeline] envVarsForTool
[Pipeline] withEnv
[Pipeline] {
[Pipeline] script
[Pipeline] {
[Pipeline] fileExists
[Pipeline] echo
Project directory does not exist. Proceeding to clone...
[Pipeline] }
[Pipeline] // script
[Pipeline] }
[Pipeline] // withEnv
[Pipeline] }
[Pipeline] // stage
[Pipeline] stage
[Pipeline] { (Clone Repository)
[Pipeline] tool
[Pipeline] envVarsForTool
[Pipeline] withEnv
[Pipeline] {
[Pipeline] git
Selected Git installation does not exist. Using Default
The recommended git tool is: NONE
No credentials specified
 > git rev-parse --resolve-git-dir /var/jenkins_home/workspace/simple-java-maven-app2/.git # timeout=10
Fetching changes from the remote Git repository
 > git config remote.origin.url https://github.com/jglick/simple-maven-project-with-tests.git # timeout=10
Fetching upstream changes from https://github.com/jglick/simple-maven-project-with-tests.git
 > git --version # timeout=10
 > git --version # 'git version 2.39.5'
 > git fetch --tags --force --progress -- https://github.com/jglick/simple-maven-project-with-tests.git +refs/heads/*:refs/remotes/origin/* # timeout=10
 > git rev-parse refs/remotes/origin/declarative^{commit} # timeout=10
Checking out Revision 88194bba5db3b3f5cd204e27c151b6788f896d9d (refs/remotes/origin/declarative)
 > git config core.sparsecheckout # timeout=10
 > git checkout -f 88194bba5db3b3f5cd204e27c151b6788f896d9d # timeout=10
 > git branch -a -v --no-abbrev # timeout=10
 > git branch -D declarative # timeout=10
 > git checkout -b declarative 88194bba5db3b3f5cd204e27c151b6788f896d9d # timeout=10
Commit message: "`label BUILD_TAG` deprecated"
 > git rev-list --no-walk 88194bba5db3b3f5cd204e27c151b6788f896d9d # timeout=10
[Pipeline] }
[Pipeline] // withEnv
[Pipeline] }
[Pipeline] // stage
[Pipeline] stage
[Pipeline] { (Build)
[Pipeline] tool
[Pipeline] envVarsForTool
[Pipeline] withEnv
[Pipeline] {
[Pipeline] sh
+ mvn -Dmaven.test.failure.ignore=true clean package
[INFO] Scanning for projects...
[INFO] 
[INFO] ----------------< test:simple-maven-project-with-tests >----------------
[INFO] Building simple-maven-project-with-tests 1.0-SNAPSHOT
[INFO]   from pom.xml
[INFO] --------------------------------[ jar ]---------------------------------
[INFO] 
[INFO] --- clean:3.2.0:clean (default-clean) @ simple-maven-project-with-tests ---
[INFO] Deleting /var/jenkins_home/workspace/simple-java-maven-app2/target
[INFO] 
[INFO] --- resources:3.3.1:resources (default-resources) @ simple-maven-project-with-tests ---
[INFO] skip non existing resourceDirectory /var/jenkins_home/workspace/simple-java-maven-app2/src/main/resources
[INFO] 
[INFO] --- compiler:3.13.0:compile (default-compile) @ simple-maven-project-with-tests ---
[INFO] No sources to compile
[INFO] 
[INFO] --- resources:3.3.1:testResources (default-testResources) @ simple-maven-project-with-tests ---
[INFO] skip non existing resourceDirectory /var/jenkins_home/workspace/simple-java-maven-app2/src/test/resources
[INFO] 
[INFO] --- compiler:3.13.0:testCompile (default-testCompile) @ simple-maven-project-with-tests ---
[INFO] Recompiling the module because of changed source code.
[INFO] Compiling 3 source files with javac [debug target 1.7] to target/test-classes
[WARNING] bootstrap class path not set in conjunction with -source 7
[WARNING] source value 7 is obsolete and will be removed in a future release
[WARNING] target value 7 is obsolete and will be removed in a future release
[WARNING] To suppress warnings about obsolete options, use -Xlint:-options.
[INFO] 
[INFO] --- surefire:2.18.1:test (default-test) @ simple-maven-project-with-tests ---
[INFO] Surefire report directory: /var/jenkins_home/workspace/simple-java-maven-app2/target/surefire-reports

-------------------------------------------------------
 T E S T S
-------------------------------------------------------
Running test.SomeTest
Tests run: 6, Failures: 0, Errors: 0, Skipped: 1, Time elapsed: 0.002 sec - in test.SomeTest
Running test.OtherTest
Tests run: 1, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0 sec - in test.OtherTest

Results :

Tests run: 7, Failures: 0, Errors: 0, Skipped: 1

[INFO] 
[INFO] --- jar:3.4.1:jar (default-jar) @ simple-maven-project-with-tests ---
[WARNING] JAR will be empty - no content was marked for inclusion!
[INFO] Building jar: /var/jenkins_home/workspace/simple-java-maven-app2/target/simple-maven-project-with-tests-1.0-SNAPSHOT.jar
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  1.807 s
[INFO] Finished at: 2025-03-01T23:49:54Z
[INFO] ------------------------------------------------------------------------
Post stage
[Pipeline] junit
Recording test results
[Checks API] No suitable checks publisher found.
[Pipeline] archiveArtifacts
Archiving artifacts
[Pipeline] }
[Pipeline] // withEnv
[Pipeline] }
[Pipeline] // stage
[Pipeline] }
[Pipeline] // withEnv
[Pipeline] }
[Pipeline] // node
[Pipeline] End of Pipeline
Finished: SUCCESS
```


![alt text](image-36.png)

Forkeamos el repo y agregamos nuestro pipeline en el Jenkinsfile

![alt text](image-37.png)

Luego usamos la opcion de Pipeline script from SCM y con el link del repo

```
Started by user admin

Obtained Jenkinsfile from git https://github.com/elkofix/simple-maven-project-with-tests
[Pipeline] Start of Pipeline
[Pipeline] node
Running on Jenkins
 in /var/jenkins_home/workspace/simple-java-maven-app2
[Pipeline] {
[Pipeline] stage
[Pipeline] { (Declarative: Checkout SCM)
[Pipeline] checkout
Selected Git installation does not exist. Using Default
The recommended git tool is: NONE
No credentials specified
 > git rev-parse --resolve-git-dir /var/jenkins_home/workspace/simple-java-maven-app2/.git # timeout=10
Fetching changes from the remote Git repository
 > git config remote.origin.url https://github.com/elkofix/simple-maven-project-with-tests # timeout=10
Fetching upstream changes from https://github.com/elkofix/simple-maven-project-with-tests
 > git --version # timeout=10
 > git --version # 'git version 2.39.5'
 > git fetch --tags --force --progress -- https://github.com/elkofix/simple-maven-project-with-tests +refs/heads/*:refs/remotes/origin/* # timeout=10
 > git rev-parse refs/remotes/origin/master^{commit} # timeout=10
Checking out Revision e687e649be5d96376437de41bed08e049397f0ba (refs/remotes/origin/master)
 > git config core.sparsecheckout # timeout=10
 > git checkout -f e687e649be5d96376437de41bed08e049397f0ba # timeout=10
Commit message: "Update Jenkinsfile"
First time build. Skipping changelog.
[Pipeline] }
[Pipeline] // stage
[Pipeline] withEnv
[Pipeline] {
[Pipeline] stage
[Pipeline] { (Declarative: Tool Install)
[Pipeline] tool
[Pipeline] envVarsForTool
[Pipeline] }
[Pipeline] // stage
[Pipeline] withEnv
[Pipeline] {
[Pipeline] stage
[Pipeline] { (Check and Clean Workspace)
[Pipeline] tool
[Pipeline] envVarsForTool
[Pipeline] withEnv
[Pipeline] {
[Pipeline] script
[Pipeline] {
[Pipeline] fileExists
[Pipeline] echo
Project directory does not exist. Proceeding to clone...
[Pipeline] }
[Pipeline] // script
[Pipeline] }
[Pipeline] // withEnv
[Pipeline] }
[Pipeline] // stage
[Pipeline] stage
[Pipeline] { (Clone Repository)
[Pipeline] tool
[Pipeline] envVarsForTool
[Pipeline] withEnv
[Pipeline] {
[Pipeline] git
Selected Git installation does not exist. Using Default
The recommended git tool is: NONE
No credentials specified
 > git rev-parse --resolve-git-dir /var/jenkins_home/workspace/simple-java-maven-app2/.git # timeout=10
Fetching changes from the remote Git repository
 > git config remote.origin.url https://github.com/jglick/simple-maven-project-with-tests.git # timeout=10
Fetching upstream changes from https://github.com/jglick/simple-maven-project-with-tests.git
 > git --version # timeout=10
 > git --version # 'git version 2.39.5'
 > git fetch --tags --force --progress -- https://github.com/jglick/simple-maven-project-with-tests.git +refs/heads/*:refs/remotes/origin/* # timeout=10
 > git rev-parse refs/remotes/origin/declarative^{commit} # timeout=10
Checking out Revision 88194bba5db3b3f5cd204e27c151b6788f896d9d (refs/remotes/origin/declarative)
 > git config core.sparsecheckout # timeout=10
 > git checkout -f 88194bba5db3b3f5cd204e27c151b6788f896d9d # timeout=10
 > git branch -a -v --no-abbrev # timeout=10
 > git branch -D declarative # timeout=10
 > git checkout -b declarative 88194bba5db3b3f5cd204e27c151b6788f896d9d # timeout=10
Commit message: "`label BUILD_TAG` deprecated"
 > git rev-list --no-walk 88194bba5db3b3f5cd204e27c151b6788f896d9d # timeout=10
[Pipeline] }
[Pipeline] // withEnv
[Pipeline] }
[Pipeline] // stage
[Pipeline] stage
[Pipeline] { (Build)
[Pipeline] tool
[Pipeline] envVarsForTool
[Pipeline] withEnv
[Pipeline] {
[Pipeline] sh
+ mvn -Dmaven.test.failure.ignore=true clean package
[INFO] Scanning for projects...
[INFO] 
[INFO] ----------------< test:simple-maven-project-with-tests >----------------
[INFO] Building simple-maven-project-with-tests 1.0-SNAPSHOT
[INFO]   from pom.xml
[INFO] --------------------------------[ jar ]---------------------------------
[INFO] 
[INFO] --- clean:3.2.0:clean (default-clean) @ simple-maven-project-with-tests ---
[INFO] Deleting /var/jenkins_home/workspace/simple-java-maven-app2/target
[INFO] 
[INFO] --- resources:3.3.1:resources (default-resources) @ simple-maven-project-with-tests ---
[INFO] skip non existing resourceDirectory /var/jenkins_home/workspace/simple-java-maven-app2/src/main/resources
[INFO] 
[INFO] --- compiler:3.13.0:compile (default-compile) @ simple-maven-project-with-tests ---
[INFO] No sources to compile
[INFO] 
[INFO] --- resources:3.3.1:testResources (default-testResources) @ simple-maven-project-with-tests ---
[INFO] skip non existing resourceDirectory /var/jenkins_home/workspace/simple-java-maven-app2/src/test/resources
[INFO] 
[INFO] --- compiler:3.13.0:testCompile (default-testCompile) @ simple-maven-project-with-tests ---
[INFO] Recompiling the module because of changed source code.
[INFO] Compiling 3 source files with javac [debug target 1.7] to target/test-classes
[WARNING] bootstrap class path not set in conjunction with -source 7
[WARNING] source value 7 is obsolete and will be removed in a future release
[WARNING] target value 7 is obsolete and will be removed in a future release
[WARNING] To suppress warnings about obsolete options, use -Xlint:-options.
[INFO] 
[INFO] --- surefire:2.18.1:test (default-test) @ simple-maven-project-with-tests ---
[INFO] Surefire report directory: /var/jenkins_home/workspace/simple-java-maven-app2/target/surefire-reports

-------------------------------------------------------
 T E S T S
-------------------------------------------------------
Running test.SomeTest
Tests run: 6, Failures: 0, Errors: 0, Skipped: 2, Time elapsed: 0.003 sec - in test.SomeTest
Running test.OtherTest
Tests run: 1, Failures: 0, Errors: 0, Skipped: 1, Time elapsed: 0.001 sec - in test.OtherTest

Results :

Tests run: 7, Failures: 0, Errors: 0, Skipped: 3

[INFO] 
[INFO] --- jar:3.4.1:jar (default-jar) @ simple-maven-project-with-tests ---
[WARNING] JAR will be empty - no content was marked for inclusion!
[INFO] Building jar: /var/jenkins_home/workspace/simple-java-maven-app2/target/simple-maven-project-with-tests-1.0-SNAPSHOT.jar
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  3.703 s
[INFO] Finished at: 2025-03-02T00:36:18Z
[INFO] ------------------------------------------------------------------------
Post stage
[Pipeline] junit
Recording test results
[Checks API] No suitable checks publisher found.
[Pipeline] archiveArtifacts
Archiving artifacts
[Pipeline] }
[Pipeline] // withEnv
[Pipeline] }
[Pipeline] // stage
[Pipeline] }
[Pipeline] // withEnv
[Pipeline] }
[Pipeline] // withEnv
[Pipeline] }
[Pipeline] // node
[Pipeline] End of Pipeline
Finished: SUCCESS
```

![alt text](image-38.png)