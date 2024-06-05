This Project shows how to use  full CI/CD pipeline using Python, GitHub, Jenkins, Docker, and deploying to another Linux host involves several steps. 


### Step 1: Create a Real-World Python Code

1. **Set up your Python project locally:**
   - Create a new directory for your project.
   - Initialize a Git repository.

   ```bash
   mkdir my-python-project
   cd my-python-project
   git init
   ```

2. **Create a simple Python application:**
   - Create a `main.py` file with a simple Python script. For example, a Flask application:

   ```python
   # main.py
   from flask import Flask

   app = Flask(__name__)

   @app.route('/')
   def hello_world():
       return 'Hello, World!'

   if __name__ == '__main__':
       app.run(host='0.0.0.0')
   ```

3. **Create a `requirements.txt` file to manage dependencies:**

   ```txt
   Flask==2.0.2
   ```

4. **Test your application locally:**

   ```bash
   pip install -r requirements.txt
   python main.py
   ```

### Step 2: Host the Project on GitHub

1. **Create a new GitHub repository:**
   - Go to GitHub and create a new repository named `my-python-project`.

2. **Push your local project to GitHub:**

   ```bash
   git remote add origin https://github.com/your-username/my-python-project.git
   git add .
   git commit -m "Initial commit"
   git push -u origin master
   ```

### Step 3: Create a Jenkins Server on Linux Machine

1. **Install Jenkins:**
   - SSH into your Linux machine and install Jenkins:

   ```bash
   sudo apt update
   sudo apt install openjdk-11-jdk -y
   wget -q -O - https://pkg.jenkins.io/debian/jenkins.io.key | sudo apt-key add -
   sudo sh -c 'echo deb http://pkg.jenkins.io/debian-stable binary/ > /etc/apt/sources.list.d/jenkins.list'
   sudo apt update
   sudo apt install jenkins -y
   sudo systemctl start jenkins
   sudo systemctl status jenkins
   ```

2. **Set up Jenkins:**
   - Open a web browser and navigate to `http://<your_server_ip>:8080`.
   - Follow the instructions to unlock Jenkins using the initial admin password.
   - Install the suggested plugins.
   - Create an admin user.

### Step 4: Create a CI Job in Jenkins

1. **Install necessary Jenkins plugins:**
   - Go to `Manage Jenkins` > `Manage Plugins` and install the following plugins:
     - Git Plugin
     - Pipeline Plugin
     - Docker Plugin
     - Docker Pipeline Plugin

2. **Create a new Jenkins pipeline job:**
   - Go to `New Item`, enter a name for your job, select `Pipeline`, and click `OK`.

3. **Configure the pipeline:**
   - In the `Pipeline` section, select `Pipeline script from SCM`.
   - Choose `Git` and enter your GitHub repository URL.
   - Define the branch as `*/master`.
   - In the `Script Path`, specify `Jenkinsfile`.

4. **Create a `Jenkinsfile` in your GitHub repository:**

   ```groovy
   pipeline {
       agent any

       stages {
           stage('Checkout') {
               steps {
                   git 'https://github.com/your-username/my-python-project.git'
               }
           }

           stage('Install Dependencies') {
               steps {
                   sh 'pip install -r requirements.txt'
               }
           }

           stage('Code Linting') {
               steps {
                   sh 'flake8 .'
               }
           }

           stage('Run Tests') {
               steps {
                   sh 'pytest'
               }
           }
       }

       post {
           always {
               junit 'reports/*.xml'
           }
           success {
               echo 'Build succeeded!'
           }
           failure {
               echo 'Build failed!'
           }
       }
   }
   ```

5. **Add dependencies for linting and testing:**
   - Update `requirements.txt`:

   ```txt
   Flask==2.0.2
   flake8==4.0.1
   pytest==7.1.2
   ```

### Step 5: Create a CD Job for Docker Image Creation and Deployment

1. **Update the `Jenkinsfile` to include Docker image creation and deployment:**

   ```groovy
   pipeline {
       agent any

       stages {
           stage('Checkout') {
               steps {
                   git 'https://github.com/your-username/my-python-project.git'
               }
           }

           stage('Install Dependencies') {
               steps {
                   sh 'pip install -r requirements.txt'
               }
           }

           stage('Code Linting') {
               steps {
                   sh 'flake8 .'
               }
           }

           stage('Run Tests') {
               steps {
                   sh 'pytest'
               }
           }

           stage('Build Docker Image') {
               steps {
                   script {
                       dockerImage = docker.build("your-dockerhub-username/my-python-project:${env.BUILD_ID}")
                   }
               }
           }

           stage('Push Docker Image') {
               steps {
                   script {
                       docker.withRegistry('https://registry.hub.docker.com', 'dockerhub-credentials') {
                           dockerImage.push("${env.BUILD_ID}")
                           dockerImage.push("latest")
                       }
                   }
               }
           }
       }

       post {
           always {
               junit 'reports/*.xml'
           }
           success {
               echo 'Build succeeded!'
           }
           failure {
               echo 'Build failed!'
           }
       }
   }
   ```

2. **Create a `Dockerfile` in your project:**

   ```Dockerfile
   FROM python:3.9-slim

   WORKDIR /app
   COPY . /app

   RUN pip install -r requirements.txt

   CMD ["python", "main.py"]
   ```

3. **Add Docker credentials to Jenkins:**
   - Go to `Manage Jenkins` > `Manage Credentials`.
   - Add a new credential of type `Username with password` for Docker Hub.

### Step 6: Deploy Docker Container to Another Linux Host

1. **Set up SSH access between Jenkins server and the target Linux host:**
   - Ensure SSH access is configured for Jenkins to communicate with the target host.

2. **Update the `Jenkinsfile` to deploy the Docker container:**

   ```groovy
   pipeline {
       agent any

       stages {
           stage('Checkout') {
               steps {
                   git 'https://github.com/your-username/my-python-project.git'
               }
           }

           stage('Install Dependencies') {
               steps {
                   sh 'pip install -r requirements.txt'
               }
           }

           stage('Code Linting') {
               steps {
                   sh 'flake8 .'
               }
           }

           stage('Run Tests') {
               steps {
                   sh 'pytest'
               }
           }

           stage('Build Docker Image') {
               steps {
                   script {
                       dockerImage = docker.build("your-dockerhub-username/my-python-project:${env.BUILD_ID}")
                   }
               }
           }

           stage('Push Docker Image') {
               steps {
                   script {
                       docker.withRegistry('https://registry.hub.docker.com', 'dockerhub-credentials') {
                           dockerImage.push("${env.BUILD_ID}")
                           dockerImage.push("latest")
                       }
                   }
               }
           }

           stage('Deploy to Host') {
               steps {
                   script {
                       sshagent(['your-ssh-credentials-id']) {
                           sh '''
                               ssh user@target-host "docker pull your-dockerhub-username/my-python-project:latest"
                               ssh user@target-host "docker stop my-python-app || true && docker rm my-python-app || true"
                               ssh user@target-host "docker run -d --name my-python-app -p 80:5000 your-dockerhub-username/my-python-project:latest"
                           '''
                       }
                   }
               }
           }
       }

       post {
           always {
               junit 'reports/*.xml'
           }
           success {
               echo 'Build succeeded!'
           }
           failure {
               echo 'Build failed!'
           }
       }
   }
   ```

### Step 7: Create a Job to Launch the Container

1. **Ensure the target host is configured correctly:**
   - Docker should be installed and running.
   - Ensure that the necessary ports are open.

2. **Test the deployment:**
   - Ensure the deployment steps in the Jenkins pipeline run correctly and the application is accessible from the target host.

### Conclusion

You have now created a complete CI/CD pipeline for a Python project hosted on GitHub, with Jenkins managing the build, test, Docker image creation, pushing to Docker Hub, and deploying to another Linux host. This setup provides a robust workflow for continuous integration and continuous deployment of your application.
