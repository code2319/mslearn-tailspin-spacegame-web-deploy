# Set the upstream remote
Run this git remote command to list your remotes:  
`$ git remote -v`

Set the remote:
```
git remote set-url origin git@github.com:code2319/mslearn-tailspin-spacegame-web-deploy.git
```

You see that you have both fetch (download) and push (upload) access to your repository:
```
origin  git@github.com:code2319/mslearn-tailspin-spacegame-web-deploy.git (fetch)
origin  git@github.com:code2319/mslearn-tailspin-spacegame-web-deploy.git (push)
``````

Origin specifies your repository on GitHub. When you fork code from another repository, the original remote (the one you forked from) is often named upstream.

Run this git remote add command to create a remote named upstream that points to the Microsoft repository:

```
$ git remote add upstream https://github.com/MicrosoftDocs/mslearn-tailspin-spacegame-web-deploy.git
```

Run git remote again to see the changes:  
`$ git remote -v`

# Select an Azure region
From the Cloud Shell, to list the regions that are available from your Azure subscription, run the following az account list-locations command.
Azure CLI

```
$ az account list-locations \
  --query "[].{Name: name, DisplayName: displayName}" \
  --output table
```

Run az configure to set your default region. Replace <REGION> with the name of the region you selected.
```
$ az configure --defaults location=northeurope
```

# Create the App Service instances
To create your App Service instances, follow these steps:

To create a resource group that's named tailspin-space-game-rg, run the following az group create command.
```
$ az group create --name tailspin-space-game-rg
```
To create an App Service plan that's named tailspin-space-game-asp, run the following az appservice plan create command.
```
$ az appservice plan create \
  --name tailspin-space-game-asp \
  --resource-group tailspin-space-game-rg \
  --sku S1 \
  --is-linux
```
The --sku argument specifies the B1 plan. This plan runs on the Basic tier. The --is-linux argument specifies to use Linux workers.

To create the three App Service instances, one for each environment (Dev, Test, and Staging), run the following az webapp create commands.
```
$ az webapp create \
  --name tailspin-space-game-web-dev-$webappsuffix \
  --resource-group tailspin-space-game-rg \
  --plan tailspin-space-game-asp \
  --runtime "DOTNET|6.0"

$ az webapp create \
  --name tailspin-space-game-web-test-$webappsuffix \
  --resource-group tailspin-space-game-rg \
  --plan tailspin-space-game-asp \
  --runtime "DOTNET|6.0"

$ az webapp create \
  --name tailspin-space-game-web-staging-$webappsuffix \
  --resource-group tailspin-space-game-rg \
  --plan tailspin-space-game-asp \
  --runtime "DOTNET|6.0"
```

For learning purposes, here, you apply the same App Service plan, B1 Basic, to each App Service instance. In practice, you would assign a plan that matches your expected workload.

For example, for the environments that map to the Dev and Test stages, B1 Basic might be appropriate because you want only your team to access the environments.

For the Staging environment, you would select a plan that matches your production environment. That plan would likely provide greater CPU, memory, and storage resources. Under the plan, you can run performance tests, like load tests, in an environment that resembles your production environment. You can run the tests without affecting live traffic to your site

To list the host name and state of each App Service instance, run the following az webapp list command.
```
$ az webapp list \
  --resource-group tailspin-space-game-rg \
  --query "[].{hostName: defaultHostName, state: state}" \
  --output table
```
Note the host name for each running service. You'll need the host names later when you verify your work.

# Create pipeline variables in Azure Pipelines
Add one variable for each App Service instance that corresponds to a Dev, Test, or Staging stage in your pipeline.
Space Game - web - Multistage > Pipelines > Library > + Variable group > + Add

For the name of your variable, enter WebAppNameDev. For the value, enter the name of the App Service instance that corresponds to your Dev environment, such as tailspin-space-game-web-dev-1234 and not tailspin-space-game-web-dev-1234.azurewebsites.net.

Repeat the previous two steps twice more to create variables for your Test and Staging environments.

# Create the dev and test environments
Azure Pipelines > Environments > Create environment

Under Name, enter dev. Leave the remaining fields at their default values. Select Create. Same for test.

# Create a service connection
Service connection enables Azure Pipelines to access your Azure subscription. Azure Pipelines uses this service connection to deploy the website to App Service. You created a similar service connection in the previous module.

Project settings > Pipelines > Service connections > New service connection > Azure Resource Manager > Service principal (automatic)

Fill in these fields:
Resource Group:	tailspin-space-game-rg
Service connection name:	Resource Manager - Tailspin - Space Game

During the process, you might be prompted to sign in to your Microsoft account.

P.S. Ensure you have selected Grant access permission to all pipelines.

# Fetch the branch from GitHub
To fetch a branch named release from the Microsoft repository, and to switch to that branch, run the following git commands.
$ git fetch upstream release
$ git checkout -B release upstream/release

The format of these commands enables you to get starter code from the Microsoft GitHub repository, known as upstream. Shortly, you'll push this branch up to your GitHub repository, known as origin.

# Promote changes to the Dev stage
1. Modify azure-pipelines.yml
2. Commit the change, and push it up to GitHub
```
$ git add azure-pipelines.yml
$ git commit -m "Deploy to the Dev stage"
$ git push origin release
```
3. In Azure Pipelines, go to the build. As it runs, trace the build.

# Create the staging environment
You can define an environment through Azure Pipelines that includes specific criteria for your release. This criteria can include the pipelines that are authorized to deploy to the environment. You can also specify the human approvals that are needed to promote the release from one stage to the next. Here, you specify those approvals.

Pipelines > Environments > New environment > Name: staging, Leave the remaining fields at their default values > Create

On the staging environment page, open the dropdown, and then select *Approvals and checks* > Approvals > Add users and groups > Save

# Clean up Azure resources
To delete the resource group:
```
az group delete --name tailspin-space-game-rg
```

As an optional step, after the previous command finishes, run the following az group list command.
```
az group list --output table
```

# Implement the blue-green deployment pattern

Run the following command to get the name of the App Service instance that corresponds to Staging and to store the result in a Bash variable that's named staging:
```
$ staging=$(az webapp list --resource-group tailspin-space-game-rg --query "[?contains(@.name, 'tailspin-space-game-web-staging')].{name: name}" --output tsv)
```

The --query argument uses [JMESPath](http://jmespath.org/), which is a query language for JSON. The argument selects the App Service instance whose name field contains "tailspin-space-game-web-staging".

Run the following command to add a slot named swap to your staging environment.
```
$ az webapp deployment slot create \
  --name $staging \
  --resource-group tailspin-space-game-rg \
  --slot swap
```

Run the following command to list your deployment slot's host name.
```
$ az webapp deployment slot list --name $staging --resource-group tailspin-space-game-rg --query [].hostNames --output tsv
```

> ! By default, a deployment slot is accessible from the internet. In practice, you could configure an Azure virtual network that places your swap slot in a network that's not routable from the internet but that only your team can access. Your production slot would remain reachable from the internet.