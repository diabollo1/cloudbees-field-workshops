---
title: "Provisioning Controllers with CloudBees CI Configuration Bundles"
chapter: false
weight: 5
--- 

The parent `base` bundle we explored in the previous lab is also configured to be the default bundle for all managed controllers that do not specify a bundle. This allows you to easily manage configuration across all of your organization's managed controllers, but it does not allow for any variations in configuration bundles between controllers. Also, the number of manual steps to provision a managed controller, and apply controller specific bundles across numerous controllers, wastes time and is prone to configuration errors. Imagine if you had dozens or even hundreds of controllers (like we do in this workshop), things would quickly become very difficult to manage.

In this lab we will explore a GitOps approach for automating the process of provisioning a controller, to include automating the configuration and assignment of a controller specific configuration bundle. This approach is based on individual repositories representing individual controllers, and takes advantage of the Jenkins GitHub Organization project type and CloudBees CI custom marker file we setup earlier. After we are done updating the `controller-casc-update` Jenkins pipeline script in our copy of the `ops-controller` repository, a new controller will be provisioned any time you add a new GitHub repository with a `controller.yaml` file to your workshop GitHub Organization. The managed controller will be provisioned with the configuration bundle from the associated GitHub repository (one other approach may be to use folders in one repository to represent each controller).

## GitOps for Controller Provisioning with CloudBees CI Configuration Bundles

Currently, automatic and dynamic provisioning of a managed controller requires running a Groovy script on CloudBees CI Operations Center. This can easily be done from a Jenkins Pipeline by leveraging the Jenkins CLI and an administrator Jenkins API token. However, for the purposes of the shared workshop environment we will be running the provisioning job from the workshop Ops controller and will leverage [CloudBees CI Cross Team Collaboration](https://docs.cloudbees.com/docs/admin-resources/latest/pipelines/cross-team-collaboration), triggering the job with the required payload from your Ops controller.

### Review the Jenkins declarative pipeline job that will be triggered by your Ops controller on the workshop Ops controller using CloudBees CI Cross Team Collaboration. 
```groovy
def event = currentBuild.getBuildCauses()[0].event
pipeline {
  agent none
  environment {
    OPS_PROVISION_SECRET = credentials('casc-workshop-controller-provision-secret')
    CONTROLLER_PROVISION_SECRET = event.secret.toString()
  }
  options { timeout(time: 10, unit: 'MINUTES') }
  triggers {
    eventTrigger jmespathQuery("controller.action=='provision'")
  }
  stages {
    stage('Provision Managed Controller') {
      agent {
        kubernetes {
          yaml libraryResource ('podtemplates/kubectl.yml')
        }
      }
      environment {
        ADMIN_CLI_TOKEN = credentials('admin-cli-token')
        GITHUB_ORGANIZATION = event.github.organization.toString().replaceAll(" ", "-")
        GITHUB_REPOSITORY = event.github.repository.toString().toLowerCase()
        GITHUB_USER = event.github.user.toString().toLowerCase()
        CONTROLLER_FOLDER = GITHUB_ORGANIZATION.toLowerCase()
        BUNDLE_ID = "${CONTROLLER_FOLDER}-${GITHUB_REPOSITORY}"
        AVAILABILITY_PATTERN = "cloudbees-ci-casc-workshop/${GITHUB_ORGANIZATION}/${GITHUB_REPOSITORY}"
      }
      when {
        triggeredBy 'EventTriggerCause'
        environment name: 'CONTROLLER_PROVISION_SECRET', value: OPS_PROVISION_SECRET
      }
      steps {
        sh "rm -rf ./${BUNDLE_ID} || true"
        sh "mkdir -p ${BUNDLE_ID}"
        sh "git clone https://github.com/${GITHUB_ORGANIZATION}/${GITHUB_REPOSITORY}.git ${BUNDLE_ID}"
      
        container('kubectl') {
          sh "kubectl cp --namespace cbci ${BUNDLE_ID} cjoc-0:/var/jenkins_home/jcasc-bundles-store/ -c jenkins"
        }
        sh '''
          curl --user "$ADMIN_CLI_TOKEN_USR:$ADMIN_CLI_TOKEN_PSW" -XPOST \
            http://cjoc/cjoc/casc-items/create-items?path=/cloudbees-ci-casc-workshop \
            --data-binary @./$BUNDLE_ID/controller.yaml -H 'Content-Type:text/yaml'
        '''
      }
    }
  }
}
```
1. The first step is to get the event payload and assign it to a global variable available to the rest of the `pipeline`: `def event = currentBuild.getBuildCauses()[0].event`. We do this outside of the declarative `pipeline` block because you cannot assign objects to variables in declarative pipeline and we need the values before we can execute a `script` block in a `stage`.
2. There is no `agent` at the global level as it would result in the unnecessary provisioning of an agent if the `when` conditions are not met.
3. An `eventTrigger` is configured to only match a JSON payload containing `controller.action=='provision'`. We wil come back to this when we update the `controller-casc-update` pipeline for your Ops controller.
4. An agent is defined for the **Provision Managed Controller** `stage` as the `sh` steps require a normal (some times referred to as heavy-weight) executor (meaning it must be run on an agent since all managed controllers our configured with 0 executors): `agent { label 'default-jnlp' }`
5. The declarative `environment` directive is used to capture values published by the Cross Team Collaboration `event`, to retrieve the controller provisioning secret value from the workshop Ops controller and to retrieve the Operations Center admin API token credential.
6. Multiple `when` conditions are configured so the **Provision Managed Controller** `stage` will only run if the job is triggered by an `EventTriggerCause` and if the `PROVISION_SECRET` matches the event payload secret.
7. The `kubectl` CLI tool is used to copy CasC bundle files to Operations Center.
8. Finally, the `curl` command is used to post the contents of the `controller.yaml` file to the `/casc-items/create-items` CloudBees CI CasC HTTP API endpoint which will result in the provisioning of a managed controller.

### Create Ops controller Job to Trigger Provisioning
Now that we have reviewed the pipeline script for the workshop provisioning of managed controllers, we need to create a new GitHub Organization Folder project that will publish a Cross Team Collaboration event to trigger that pipeline.

1. Navigate to your copy of the `ops-controller` repository in GitHub and click on the **Add file** button and then select **Create new file**. ![Create new file in GitHub](github-create-new-file-ops.png?width=50pc)
2. Name the new file `controller-provision`, copy the `controller-provision` contents from below and paste it into the GitHub file editor. Notice that the `controller-provision` Declarative Pipeline script is loading a `credential` with an id of `casc-workshop-controller-provision-secret`.

```groovy
library 'pipeline-library'
pipeline {
  agent any
  options {
    timeout(time: 10, unit: 'MINUTES')
  }
  stages {
    stage('Publish Provision Controller Event') {
      when {
        branch 'main'
      }
      environment { PROVISION_SECRET = credentials('casc-workshop-controller-provision-secret') }
      steps {
        gitHubParseOriginUrl()
        publishEvent event:jsonEvent("""
          {'controller':{'name':'${GITHUB_REPO}','action':'provision'},'github':{'organization':'${GITHUB_ORG}','repository':'${GITHUB_REPO}','user':'${GITHUB_USER}'},'secret':'${PROVISION_SECRET}'}
        """), verbose: true
      }
    }
  }
}
```

3. Next, select the option to **"Create a new branch for this commit and start a pull request"**, name the branch `add-provision-job` and finally click the **Propose new file** button. ![Add provision job](add-provision-job.png?width=50pc)
4. On the next screen click the **Create pull request** button to create a pull request to merge to the `main` branch when you are done updating your `ops-controller` configuration bundle. ![Create add provision job pull request](github-create-provision-pr.png?width=50pc)
5. Next, navigate back to the **Code** tab of your `ops-controller` repository and select the `add-provision-job` branch from the branch drop down.
6. Back at the top level of your `ops-controller` repository, click on the `jcasc` folder, then click on the `credentials.yaml` file and then click on the ***Edit this file*** pencil button to edit the file. 
7. Add the `casc-workshop-controller-provision-secret` below the top `credentials` entry:

```yaml

  restrictedSystem:
    domainCredentials:
    - allowList: "controller-jobs/controller-provision/**/main"
      credentials:
        string:
          description: "CasC Workshop Controller Provision Secret"
          id: "casc-workshop-controller-provision-secret"
          scope: GLOBAL
          secret: "${cbciCascWorkshopControllerProvisionSecret}"
```

{{%expand "expand for complete updated credentials.yaml file" %}}
```yaml
credentials:
  restrictedSystem:
    domainCredentials:
    - allowList: "controller-jobs/controller-provision/**/main"
      credentials:
        string:
          description: "CasC Workshop Controller Provision Secret"
          id: "casc-workshop-controller-provision-secret"
          scope: GLOBAL
          secret: "${cbciCascWorkshopControllerProvisionSecret}"
  system:
    domainCredentials:
    - credentials:
      - string:
          description: "CasC Update Secret"
          id: "casc-update-secret"
          scope: GLOBAL
          secret: "${cbciCascWorkshopControllerProvisionSecret}"
      - string:
          description: "Webhook secret for CloudBees CI Workshop GitHub App"
          id: "cloudbees-ci-workshop-github-webhook-secret"
          scope: SYSTEM
          secret: "${gitHubWebhookSecret}"
      - gitHubApp:
          apiUri: "https://api.github.com"
          appID: "${cbciCascWorkshopGitHubAppId}"
          description: "CloudBees CI CasC Workshop GitHub App credential"
          id: "cloudbees-ci-casc-workshop-github-app"
          owner: "${GITHUB_ORGANIZATION}"
          privateKey: "${cbciCascWorkshopGitHubAppPrivateKey}"
```
{{% /expand%}}

8. After you have pasted the new credential into the `credentials.yaml` file, ensure that you are committing to the `add-provision-job` branch and then click the **Commit new file** button. ![Commit credentials.yaml](github-commit-credentials-yaml.png?width=50pc)
9. Navigating back to the top level of your `ops-controller` repository and ensuring that you are on the `add-provision-job` branch, click on the `items.yaml` file and then click on the ***Edit this file*** pencil button to edit the file. 
10. Copy the entire `controller-casc-update` item and paste it below the second `items` entry (it should be line 19).
11. Change the `name` of the new `organizationFolder` item to `controller-provision`, change the value for `marker` to `controller.yaml` and change the `scriptPath` to `controller-provision`. 
12. After you have made those changes, ensure that you are committing to the `add-provision-job` branch and then click the **Commit new file** button. ![Commit items.yaml](github-commit-items-yaml.png?width=50pc)
13. We have now made all the necessary changes and can now merge the pull request to the `main` branch. In GitHub, click on the **Pull requests** tab and then click on the link for the **Create controller-provision** pull request.
14. On the **Create controller-provision #3** pull request page, click the **Merge pull request** button and then click the **Confirm merge** button.
15. Navigate to the `main` branch job of the `controller-casc-update` `ops-controller` Multibranch pipeline project on your Ops controller. ![ops-controller Mulitbranch](ops-controller-multibranch-jcasc.png?width=50pc)
16. After the the `main` branch job has completed successfully, navigate to the `controller-jobs` folder and refresh the page until the `controller-provision` job appears. ![controller-provision-job](controller-provision-job.png?width=50pc)

### Create a new managed controller repository
In the previous section you added the `controller-provision` Organization Folder job to your Ops controller to publish an event to trigger the provisioning of a managed controller. Now you will trigger the provisioning of a new managed controller by adding a `controller.yaml` file to the `main` branch of your copy of the `dev-controller` repository in your workshop GitHub Organization.

1. Navigate to your copy of the `dev-controller` repository in GitHub and click on the **Pull requests** link and then click on the link for the **Provision Controller** pull request.
2. On the next screen, click on the **Files changed** tab to review the `bundle.yaml` and `controller.yaml` files being added to your `dev-controller` repository. Note that the `bundle.yaml` file does include any other configuration files but it does specify `parent: base`.
3. Once you have reviewed the changed files, click on the **Conversation** tab, scroll down and click the green **Merge pull request** button and then the **Confirm merge** button.
4. Next, navigate to the top level of your Ops controller, click the on the `controller-jobs` folder and then click on the `controller-provision` job. There should now be a `dev-controller` folder under the `controller-provision` job for your 
`dev-controller` repository. ![dev-controller job](dev-controller-job.png?width=50pc)
5. Navigate to the `main` branch of the **dev-controller** Multi-branch pipeline job on your Ops controller and you should see an **Event JSON** in the build logs similar to the one below (of course the GitHub information will be unique to you). ![Event JSON log output](event-json-log-output.png?width=50pc)
6. A few minutes after the **dev-controller** **main** branch job completes you will have a new managed controller named **dev-controller** in the same folder as your Ops controller. ![New dev-controller](new-dev-controller.png?width=50pc)
