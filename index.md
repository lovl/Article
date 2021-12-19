Build and deploy an ASP.NET core app running on IIS using Azure pipelines

Tai Bo
Aug 10, 2019·12 min read

Image by Dirk Wouters from Pixabay
This is part III of the series in which I cover how to build and deploy an ASP.NET core app to IIS running on a Windows VM. In part I, I cover how to build and publish an artifact using visual studio. In part II, I cover how to manually deploy the artifact to IIS and go over some of the concepts such as bindings, app pools and website.
In this tutorial, I walks you through the steps to automate the build and deployment process using azure pipelines.
Assumptions
The machine to which you want to deploy your ASP.NET core application should have IIS enabled. If your app specifies framework dependent as the deployment mode, the server to which you deploy the application should have the appropriate .NET core runtime installed.
You have remote access to the remote servers.
You use git as your source control management system.
I. Account
You can get started with Azure Devops Services for free. The free plan gives you access to a Microsoft hosted server for running jobs to build and deploy your projects. You get 1,800 minutes per month and can run one job at a time. For more details, checkout the document. If you don’t have an account yet, go to this link to register.
II. Project
Before you can create a pipeline, you need to create a project. A project is more or less a container which isolates features and services from one project to another. A project consists of one or more repositories, project management tools, pipelines, hosted nuget packages etc..
Each project you create provides boundaries to isolate data from other projects and must be managed and structured to support your business needs.
Source: Create a project in Azure DevOps and TFS
In azure devops, click “Create project” button to create a project.
For this tutorial, I have scaffolded a simple ASP.NET core project using Visual Studio. You can do the same if you want to follow alone. Checkout this quick start guide which goes over how to create an ASP.NET core app in Visual Studio.
Creating a new project in azure devops
III. Source codes
Azure devops need to know where to grab the codes to build your project. Follow the steps below to add or connect your existing repository.
1. In azure devops, go to project under the Projects tab. If you have just created a new project, you may already be at the project page.
2. Click on Repos in the left menu.
3. Choose one of the available options.
Azure devops provide some guidance for pushing code into a repository. If you are not familiar with Git, I suggest checking out the document .
Clone to local computer -> Use the git clone command to create and initialize a local git repository that links to the remote repository.
Tais-MacBook-Pro:personal taibo$ git clone https://taithienbo@dev.azure.com/taithienbo/AspNetCoreExample/_git/AspNetCoreExampleCloning into ‘AspNetCoreExample’…warning: You appear to have cloned an empty repository.
Push an existing repository from command line -> Choose this if you already have a local git repo.
Tais-MacBook-Pro:AspNetCoreExample taibo$ git remote add origingit@ssh.dev.azure.com:v3/taithienbo/AspNetCoreExample/AspNetCoreExample
Tais-MacBook-Pro:AspNetCoreExample taibo$ git add .Tais-MacBook-Pro:AspNetCoreExample taibo$ git commit[master (root-commit) ded1fc7] checking in sample project scaffoldedby Visual Studio.Tais-MacBook-Pro:AspNetCoreExample taibo$ git push --set-upstreamorigin master
Import a repository -> If you host your git repository on Github or some other hosting providers, import the repository into azure devops to use the pipelines.
Once you have checked in the project, it’s time to build and deploy.
IV. Build pipeline
A. Create a new pipeline with defaults
1. On the left menu, click on Pipelines.
2. Click on Build under Pipelines.
3. Click New pipeline
4. Follow the steps to generate a yaml file which describes the build steps and commit the file to your repository.
Select the source repository where you host your project. For this document, I select Azure Repos Git.
Under “Select a repository”, select AspNetCoreExample.
Under “Configure your pipeline” , select ASP.NET core.Azure generates a YAML file which describes the steps for building a simple ASP.NET core app.
Commit the file to your source repository by clicking the “Save and run” button.
Click “Save and run” again. Azure devops commits and pushes the script to your repository. Then it starts the build.
B. Edit a pipeline
The default script builds the project and produces .dll binary files. However, a few more things we need to do:
1. Run dotnet publish to produce other assets such as the project’s dependencies, appsettings files, static files and output them all to a directory ready for deployment.
2. Make the artifact available at a shared location on azure devops for using in a release pipeline later.
Azure devops has built in tasks for different purposes. For example, we can use the DotNetCoreCLI task to publish the binary files into a folder for deployment and PublishBuildArtifact task to copy the folder into a shared location accessible by the release pipeline.
1. Go to the pipeline we just created, click on the “Edit” button.
2. To append new content, place cursor at the end of the yaml file.
3. Add the following snippets, then click “save”.
- task: DotNetCoreCLI@2
 inputs:
   command: 'publish'
   publishWebProjects: true
   zipAfterPublish: true
   arguments: '--output $(build.artifactstagingdirectory)'
- task: PublishBuildArtifacts@1
 inputs:
   pathToPublish: $(Build.ArtifactStagingDirectory)
   artifactName: AspNetCoreExample
In the above snippets:
We use the — output flag to specify $(build.artifactstagingdirectory) as the directory where we publish the binary files. This is the directory where the release pipeline looks for the artifact.
We make the artifact available for use later in a release pipeline, using PublishBuildArtifacts@1 built in task.
V. Release pipeline
The goal of a release pipeline is to take the published artifact from the build pipeline and deploy it on the target servers.
Before setting up the pipeline, you need to register the servers into a deployment pool and create deployment groups.
Deployment pool vs deployment group
A deployment pool is a grouping of servers to where you want to deploy an application. From a deployment pool, you can create multiple deployment groups which you can assign to projects. A deployment group is essentially the same as that of the pool from which it references. However, a deployment pool exists at organization level whereas a deployment group exists at project level.
The relationship between Deployment Pools and Deployment Groups is similar to the relationship between Agent Pools and Agent Queues. A Deployment Pool exists at the account level and is the actual container of the deployment agents (targets), whereas the Deployment Group is a layer over it which makes these targets available to release definitions in a project.
Source: Sharing of Deployment Groups across projects
To learn more about deployment group and deployment pool, checkout the documentation.
Create a deployment pool and deployment group
Before you can create a deployment pool, verify that your account has the appropriate permissions. If you have been following the steps, you probably have permissions since you are the owner of the organization. If you are not sure, checkout this document.
Now that you’ve verified the permissions, let’s create a deployment pool.
Generate agent creation script
1. Click on Azure DevOps logo
2. Select Organization settings
3. Select “Deployment pools”
4. Enter a name. (e.g. “AspNetCoreExample app servers”)
5. Under Provision a corresponding deployment group for the selected projects, check the project you created (e.g. AspNetCoreExample).
6. Click Create.
At this point, Azure devops generate a registration script. You need to take this script and run it on each of the servers where you want to deploy the app.
Register servers into deployment pools
1. Run the script on each of the target machines where you want to deploy the app.
2. Under Type of target to register, select Windows.
3. Check Use a personal access token in the script for authentication
4. Copy the script into the clipboard.
5. Connect remotely to the server using remote desktop.
6. On the server, open PowerShell in Administrator mode.
7. Paste the script into Powershell and run.
8. When prompted for a user account to use for the server, use the default NT AUTHORITY\SYSTEM.
The personal access token is how the agent (the server where you host the app) communicates securely with azure devops. The agent periodically pulls for job requests. When a job comes into the queue, the agent downloads the job and acts on it (i.e. deploy the app to IIS).
“Periodically, the agent checks to see if a new job request has been posted for it in the job queue in Azure Pipelines/TFS. When a job is available, the agent downloads the job as well as a job-specific OAuth token. This token is generated by Azure Pipelines/TFS for the scoped identity specified in the pipeline. That token is short lived and is used by the agent to access resources (e.g., source code) or modify resources (e.g., upload test results) on Azure Pipelines or TFS within that job.”
Source: Communication with Azure pipelines
The process involves connecting to the server and could take a minute.
After the script has finished successfully, go back to Deployment pools under Organization Settings. You should see the status indicate the agent is online and one project is referencing the pool.
Repeat the above steps if you have other servers you want to add into the pool.
Now that we have installed the agents on the deployment servers, register the servers into a deployment pool, and created a deployment group out of the pool for using in the project. Let’s create a release pipeline to deploy the artifact onto IIS.
Creating a release pipeline
1. Click on Releases on the left menu
2. Click on New pipeline
3. On the top, hover over “New release pipeline” and click to change the name of the pipeline (e.g. AspNetCoreExample release pipeline).
4. In the search box, enter IIS website deployment. Select the template and click Apply
5. Under Artifacts, click Add
6. Under Source (build pipeline), select the build pipeline which you created earlier (e.g. AspNetCoreExample and click Add
7. Click the + button to add a stage
8. Specify a name under Stage name (e.g. Development).
9. Click on the job and tasks link under the stage to edit the tasks.
10. Under Tasks, click on Deployment process and fill in the information on the right
11. Under Tasks, click on IIS Deployment and fill in the information on the right. Under Deployment group, select the group created earlier (e.g. AspNetCoreExample-AspNetCoreExample app servers). For other fields, leave the defaults.
12. Under Tasks, click on IIS Web App Manage, and fill in the information in the right.
13. Under Tasks, click on IIS Web App Deploy. Review the information. You actually don’t need to change anything here.
14. Click Save to create the pipeline.
For step 10, enter the following:
Stage name: Development
Configuration type: IIS website
Action: Create Or Update
Website name: MyAspNetCoreApp
Bindings:
You can edit the bindings if you have a DNS setup for your app.
You need to edit the bindings if you already have a default app running on http port 80. Otherwise, you need to stop the other app to avoid conflict. IIS needs to be able to deterministically route requests to a single app.
For step 12, use the following:
Under Physical path, replace wwwroot with a name for your app (e.g. AspNetCoreExample).
Click on IIS Web App Manage
Check Create or update app pool
Enter a name for the app pool (e.g. AspNetCoreExample)
For .NET version, select No managed code
Leave other fields default
Azure devops packs with templates for deploying different types of apps to different environments. Several of them are for deploying to IIS. The one we use is “IIS website deployment”.
A stage is an organizational concept which group together a list of jobs. A job in turn groups together a list of steps to deploy the app. You can specify what happen before and after the deployment reaches the stage. One way I use stage is to model deployment to multiple environments. For instance, I have stages that group together the steps to deploy to development, staging and production servers.
Now that we have created the release pipeline, let’s create a release and start deploying the application.
A release is the package or container that holds a versioned set of artifacts specified in a release pipeline in your DevOps CI/CD processes. It includes a snapshot of all the information required to carry out all the tasks and actions in the release pipeline, such as the stages, the tasks for each one, the values of task parameters and variables, and the release policies such as triggers, approvers, and release queuing options. There can be multiple releases from one released pipeline, and information about each one is stored and displayed in Azure Pipelines for the specified retention period.
Source: Releases in Azure Pipelines
Create a release
1. Go to the project. (e.g. AspNetCoreExample).
2. On the left menu, click Releases
3. Select the release pipeline you created.
4. On the right, click Create a release
5. Unless you make changes, by default, upon creation, azure pipeline initiates the job to automatically deploy the artifact to the target servers. In this case, we want to automatically deploy the release to the server, so just leave the defaults
6. Click Create
Azure devops queues the deployment. You can check the status by clicking on the pipeline under Releases.
For example, the picture below shows a failed deployment. I can click on the link to view details and logs.
In this case, the deployment failed because IIS is not enabled on the target server.
Cannot find appcmd.exe location. Verify IIS is configured on EC2AMAZ-BTNDA0E and try operation again.
You can deploy the same release multiple times. For instance, after making changes on the server, I retry the deployment by going to the release and clicking on the “Deploy” button.
If everything finishes successfully, you get all green checks.
On IIS, you should see the website and app pool setup, and the app’s files are installed under the expected directory (e.g. c:\ietpub\aspnetcoreexample).
Under Browse Website, click on the link to open your app. If everything is working, you see your website:
Congratulations !!!. You have successfully deployed an asp.net core application to IIS using azure pipelines.
VI. Continuous Integration
The idea of continuous integration is to constantly run the build and hence automated tests to detect issues as early as possible.
Azure devops has support for continuous integration. For instance, you can configure the build to run automatically on changes to the source code, on a scheduled, or when another build completes.
You can configure CI in the yaml file or via the interface. Microsoft has good documentation on how to setup continuous integration for your build pipeline.
VII. Continuous Deployment
The idea of continuous deployment is you have the process setup such that you can deploy an artifact to targeted servers automatically or at the click of a button.
You can trigger the release pipeline to automatically deploy whenever the build publishes a new artifact, or on schedule.
1. Go to your release pipeline and click on “Edit”.
2. Under “Stages”, click on “Pre-deployment conditions”.
3. On the right side, you can see the options to trigger the release.
4. Make appropriate selections and click “Save”.
VIII. Common errors
Following are some errors you may encounter while setting up pipelines and deployment.
Cannot find appcmd.exe. Verify IIS is configured …
This indicates you need to enable IIS on the target servers.
Binding (http / * : 80 : ) already exists for a different website … change the port and retry the operation.
This can happen if you use the default binding and you have not stopped the default website on IIS, which also uses the default binding. The default binding only specifies port 80 and no host. IS needs to be able to determine exactly the website to which it should forward a request. As such, the bindings for the site should be different from one another. If two sites have the same binding, then only one can run at a time. You can fix this by adding a host name, change a port, or stop the default website.
No package found with specified pattern.
Azure pipelines could not find the artifact. Go back to the build pipeline, check the folder where you output the build files and make sure the directory is where the release pipeline looks for the artifact. In particular, I find it is necessary to specify the outputDir for the publish command like below:
arguments: ‘ — output $(build.artifactstagingdirectory)’
Also,make sure you have the — task: PublishBuildArtifacts@1 at the end of the build yaml file.
HTTP Error 500.19 — Internal Server Error
The requested page cannot be accessed because the related configuration data for the page is invalid.
This can happen if the server does not have appropriate .net core runtime installed. Download and install the latest .NET core runtime and hosting bundle. Then restart IIS. For more information, checkout the
