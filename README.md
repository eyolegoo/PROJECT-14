# PROJECT-14
EXPERIENCE CONTINUOUS INTEGRATION WITH JENKINS | ANSIBLE | ARTIFACTORY | SONARQUBE | PHP

## ***SIMULATING A TYPICAL CI/CD PIPELINE FOR A PHP BASED APPLICATION***

- As part of the ongoing infrastructure development with Ansible started from Project 11, you will be tasked to create a pipeline that simulates continuous integration and delivery. Target end to end CI/CD pipeline is represented by the diagram below. It is important to know that both **Tooling and TODO** Web Applications are based on an interpreted [scripting](https://en.wikipedia.org/wiki/Scripting_language) language (PHP). It means, it can be deployed directly onto a server and will work without compiling the code to a machine language.

- The problem with that approach is, it would be difficult to package and version the software for different releases. And so, in this project, we will be using a different approach for releases, rather than downloading directly from git, we will be using Ansible [uri module](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/uri_module.html).

<img width="588" alt="PIPELINE SCHEME" src="https://github.com/eyolegoo/PROJECT-14/assets/115954100/6ce9626c-8e15-4f47-b730-35dcf0f5adc1">

**SET UP**

- This project is partly a continuation of your Ansible work, so simply add and subtract based on the new setup in this project. It will require a lot of servers to simulate all the different environments from *dev/ci* all the way to *production*. This will be quite a lot of servers altogether (But you don’t have to create them all at once. Only create servers required for an environment you are working with at the moment. For example, when doing deployments for development, do not create servers for integration, pentest, or production yet).

- Try to utilize your AWS free tier as much as you can, you can also register a new account if you have exhausted the current one. Alternatively, you can use [Google Cloud (GCP)](https://cloud.google.com/) to rent virtual machines from this cloud service provider – you can get $300 credit [here](https://clozon.com/try-google-cloud-services-and-get-300-credit-with-a-12-month-free-trial/) or [here](https://www.startups.com/products/benefits/googlecloud) (NOTE: Please read instructions carefully to get your credits)

- **NOTE**: This is still NOT a cloud-focus project yet. AWS cloud end to end project begins from project-15. Therefore, do not worry about advanced AWS or GCP configuration. All we need here is virtual machines that can be accessed over **SSH**.

- To minimize the cost of cloud servers, you don not have to create all the servers at once, simply spin up a minimal server set up as you progress through the project implementation and have reached a need for more.

- To get started, we will focus on these environments initially.

  - Ci
  - Dev
  - Pentest

- Both SIT – For System Integration Testing and UAT – User Acceptance Testing do not require a lot of extra installation or configuration. They are basically the webservers holding our applications. But **Pentest** – For Penetration testing is where we will conduct security related tests, so some other tools and specific configurations will be needed. In some cases, it will also be used for **Performance and Load testing**. Otherwise, that can also be a separate environment on its own. It all depends on decisions made by the company and the team running the show.

- What we want to achieve, is having Nginx to serve as a [reverse proxy](https://en.wikipedia.org/wiki/Reverse_proxy) for our sites and tools. Each environment setup is represented in the below table and diagrams.

<img width="535" alt="Environment setup" src="https://github.com/eyolegoo/PROJECT-14/assets/115954100/c9104b3b-a376-442a-82cb-760c3dc3dcf2">

- CI-Environment

<img width="398" alt="CI environment" src="https://github.com/eyolegoo/PROJECT-14/assets/115954100/3e461d84-13ce-4f92-b428-12fa206813be">

- Other Environments from Lower To Higher
  
<img width="410" alt="other environment" src="https://github.com/eyolegoo/PROJECT-14/assets/115954100/10f72814-a57d-4ed6-b590-1f3f4dbab282">

- Ansible Inventory should look like this

```
├── ci
├── dev
├── pentest
├── pre-prod
├── prod
├── sit
└── uat
```

ci inventory file

```
[jenkins]
<Jenkins-Private-IP-Address>

[nginx]
<Nginx-Private-IP-Address>

[sonarqube]
<SonarQube-Private-IP-Address>

[artifact_repository]
<Artifact_repository-Private-IP-Address>
```

dev Inventory file

```
[tooling]
<Tooling-Web-Server-Private-IP-Address>

[todo]
<Todo-Web-Server-Private-IP-Address>

[nginx]
<Nginx-Private-IP-Address>

[db:vars]
ansible_user=ec2-user
ansible_python_interpreter=/usr/bin/python

[db]
<DB-Server-Private-IP-Address>
```

pentest inventory file

```
[pentest:children]
pentest-todo
pentest-tooling

[pentest-todo]
<Pentest-for-Todo-Private-IP-Address>

[pentest-tooling]
<Pentest-for-Tooling-Private-IP-Address>
```

**Observations:**

- 1. You will notice that in the *pentest inventory file*, we have introduced a new concept *pentest:children* This is because, we want to have a group called pentest which covers Ansible execution against both *pentest-todo and pentest-tooling* simultaneously. But at the same time, we want the flexibility to run specific Ansible tasks against an individual group.
 
- 2. The db group has a slightly different configuration. It uses a RedHat/Centos Linux distro. Others are based on Ubuntu (in this case user is ubuntu). Therefore, the user required for connectivity and path to python interpreter are different. If all your environment is based on Ubuntu, you may not need this kind of set up. Totally up to you how you want to do this. Whatever works for you is absolutely fine in this scenario.
 
- This makes us to introduce another Ansible concept called *group_vars*. With group vars, we can declare and set variables for each group of servers created in the inventory file.

- For example, If there are variables we need to be common between both *pentest-todo* and *pentest-tooling*, rather than setting these variables in many places, we can simply use the *group_vars* for *pentest*. Since in the inventory file it has been created as *pentest:children* Ansible recognizes this and simply applies that variable to both children.


## ***ANSIBLE ROLES FOR CI ENVIRONMENT***

- Now go ahead and Add two more roles to ansible:

- 1. [SonarQube](https://www.sonarsource.com/products/sonarqube/) (Scroll down to the Sonarqube section to see instructions on how to set up and configure SonarQube manually)
       
- 2. [Artifactory](https://jfrog.com/artifactory/)
 
- ***Why do we need SonarQube?***

- SonarQube is an open-source platform developed by SonarSource for continuous inspection of code quality, it is used to perform automatic reviews with static analysis of code to detect bugs, [code smells](https://en.wikipedia.org/wiki/Code_smell), and security vulnerabilities. [Watch a short description here](https://www.youtube.com/watch?v=vE39Fg8pvZg). There is a lot more hands on work ahead with SonarQube and Jenkins. So, the purpose of SonarQube will be clearer to you very soon.

- ***Why do we need Artifactory?***

- Artifactory is a product by [JFrog](https://jfrog.com/) that serves as a binary repository manager. The binary repository is a natural extension to the source code repository, in that the outcome of your build process is stored. It can be used for certain other automation, but we will it strictly to manage our build artifacts.

- [Watch a short description here](https://www.youtube.com/watch?v=upJS4R6SbgM) Focus more on the first 10.08 mins

-  ***Configuring Ansible For Jenkins Deployment***

-  In previous projects, you have been launching Ansible commands manually from a CLI. Now, with Jenkins, we will start running Ansible from Jenkins UI.

-  To do this,

    - Navigate to Jenkins URL

    - Install & Open Blue Ocean Jenkins Plugin

 - Create a new pipeline

<img width="557" alt="1" src="https://github.com/eyolegoo/PROJECT-14/assets/115954100/74bd3ba8-30f7-48e6-95b5-c7dcde168ad8">

 - Select Github

<img width="581" alt="2" src="https://github.com/eyolegoo/PROJECT-14/assets/115954100/bf44758f-5e0b-4b54-adf1-f1bef8e1ba5e">

 - Connect Jenkins with GitHub

<img width="565" alt="3" src="https://github.com/eyolegoo/PROJECT-14/assets/115954100/c9a089ce-bdf0-45a7-8fa3-6d8dd12a1f02">

 - Login to GitHub & Generate an Access Tokenhttps://www.dareyio.com/wp-content/uploads/2021/07/Jenkins-Create-Access-Token-To-Github.png

<img width="556" alt="4" src="https://github.com/eyolegoo/PROJECT-14/assets/115954100/2bf304aa-42e0-4b24-a6a7-0218e259861d">

<img width="543" alt="5" src="https://github.com/eyolegoo/PROJECT-14/assets/115954100/50837e44-cb0d-42a9-9478-5dc2b90596ab">

 - Copy Access Token

<img width="591" alt="6" src="https://github.com/eyolegoo/PROJECT-14/assets/115954100/d45b69bb-2172-4d81-aa13-9238bd466779">

 - Paste the token and connect

<img width="570" alt="7" src="https://github.com/eyolegoo/PROJECT-14/assets/115954100/968e6783-e6c7-47a4-a494-fe1797160204">

 - Create a new Pipeline

<img width="580" alt="8" src="https://github.com/eyolegoo/PROJECT-14/assets/115954100/f3a3f432-fb1a-4ce6-ad8e-9ff931dc5336">

- At this point you may not have a [Jenkinsfile](https://www.jenkins.io/doc/book/pipeline/jenkinsfile/) in the Ansible repository, so Blue Ocean will attempt to give you some guidance to create one. But we do not need that. We will rather create one ourselves. So, click on Administration to exit the Blue Ocean console.

<img width="571" alt="9" src="https://github.com/eyolegoo/PROJECT-14/assets/115954100/79dfe111-f3ff-4b94-9004-106658fcb7c7">

- Here is our newly created pipeline. It takes the name of your GitHub repository.

<img width="595" alt="10" src="https://github.com/eyolegoo/PROJECT-14/assets/115954100/4c3d51ae-6496-4b41-9598-9ef34089be82">

***Let us create our*** **Jenkinsfile**

Inside the Ansible project, create a new directory **deploy** and start a new file **Jenkinsfile** inside the directory.

<img width="328" alt="Jenkinsfile" src="https://github.com/eyolegoo/PROJECT-14/assets/115954100/6733042f-7aa8-4aae-9780-fa61bf401358">

- Add the code snippet below to start building the **Jenkinsfile** gradually. This pipeline currently has just one stage called **Build** and the only thing we are doing is using the **shell script** module to echo **Building Stage**

```
pipeline {
    agent any

  stages {
    stage('Build') {
      steps {
        script {
          sh 'echo "Building Stage"'
        }
      }
    }
    }
}
```

- Now go back into the Ansible pipeline in Jenkins, and select configure

<img width="610" alt="11" src="https://github.com/eyolegoo/PROJECT-14/assets/115954100/691a573c-c4af-4a2c-a04f-068044dc83ce">

- Scroll down to **Build Configuration** section and specify the location of the Jenkinsfile at **deploy/Jenkinsfile**

<img width="538" alt="12" src="https://github.com/eyolegoo/PROJECT-14/assets/115954100/e706f543-12b0-4166-b9c3-b256820f0367">

- Back to the pipeline again, this time click "Build now"

<img width="575" alt="13" src="https://github.com/eyolegoo/PROJECT-14/assets/115954100/d6015e15-58ce-40b7-86c9-11db81c69db4">

- This will trigger a build and you will be able to see the effect of our basic **Jenkinsfile** configuration by going through the console output of the build.

- To really appreciate and feel the difference of Cloud Blue UI, it is recommended to try triggering the build again from Blue Ocean interface.

- 1. Click on Blue Ocean
 
<img width="572" alt="14" src="https://github.com/eyolegoo/PROJECT-14/assets/115954100/bcd39c2a-f1b4-4f6f-ab50-6e19c27223a6">

- 2. Select your project

- 3. Click on the play button against the branch

<img width="574" alt="15" src="https://github.com/eyolegoo/PROJECT-14/assets/115954100/eb59e767-f58d-4b53-8328-5c589b26488d">

- Notice that this pipeline is a multibranch one. This means, if there were more than one branch in GitHub, Jenkins would have scanned the repository to discover them all and we would have been able to trigger a build for each branch.

Let us see this in action.

  - Create a new git branch and name it **feature/jenkinspipeline-stages**

  - Currently we only have the **Build** stage. Let us add another stage called **Test**. Paste the code snippet below and push the new changes to GitHub.

```
pipeline {
    agent any

  stages {
    stage('Build') {
      steps {
        script {
          sh 'echo "Building Stage"'
        }
      }
    }

    stage('Test') {
      steps {
        script {
          sh 'echo "Testing Stage"'
        }
      }
    }
    }
}


```

- 4. To make your new branch show up in Jenkins, we need to tell Jenkins to scan the repository.
 
   - Click on the "Administration" button
 
<img width="554" alt="16" src="https://github.com/eyolegoo/PROJECT-14/assets/115954100/80f6f6d0-2f39-490a-bd92-cbbbac6c9003">

   - Navigate to the Ansible project and click on "Scan repository now"

<img width="567" alt="17" src="https://github.com/eyolegoo/PROJECT-14/assets/115954100/9de1c6f7-9ec2-4cc4-80e9-8d9467bdab61">

   - Refresh the page and both branches will start building automatically. You can go into Blue Ocean and see both branches there too.

<img width="557" alt="18" src="https://github.com/eyolegoo/PROJECT-14/assets/115954100/f351b493-d2fd-423a-9b89-4339085d86ab">

  - In Blue Ocean, you can now see how the Jenkinsfile has caused a new step in the pipeline launch build for the new branch.

<img width="559" alt="19" src="https://github.com/eyolegoo/PROJECT-14/assets/115954100/26377863-d403-4e28-ab08-5a7f56865f0f">

- ***A QUICK TASK FOR YOU!***

```
1. Create a pull request to merge the latest code into the main branch
2. After merging the PR, go back into your terminal and switch into the main branch.
3. Pull the latest change.
4. Create a new branch, add more stages into the Jenkins file to simulate below phases. (Just add an echo command like we have in build and test stages)
   1. Package 
   2. Deploy 
   3. Clean up
5. Verify in Blue Ocean that all the stages are working, then merge your feature branch to the main branch
6. Eventually, your main branch should have a successful pipeline like this in blue ocean
```

<img width="588" alt="20" src="https://github.com/eyolegoo/PROJECT-14/assets/115954100/0364f19e-252d-4aa9-9ad9-1b8e471fb5e9">
