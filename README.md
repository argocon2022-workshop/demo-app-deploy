# Demo Appication Deployment

The repository contains Kubernetes deployment manifests that can be used to deploy ArgoCon 2022 workshop [demo application](https://github.com/argocon2022-workshop/demo-app).
Check the [https://github.com/argocon2022-workshop](https://github.com/argocon2022-workshop) for more information.


The repository uses [Kustomize](https://kustomize.io) to define the manifests and contains the following directories:

* `base` - contains the base manifests that are used to deploy the application
    * `deployment.yaml` - defines the deployment
    * `service.yaml` - defines the service
* `env` - contains the environment specific overlays
    * `dev`
    * `stage`
    * `prod`


## Preparation

Please perform the following steps to start using the repository:

* [Fork](https://github.com/argocon2022-workshop/demo-app-deploy/fork) it to your own GitHub account.
* Enable Github Actions for the repository:
    * Navigate to `https://github.com/<USERNAME>/demo-app-deploy/settings/secrets/actions`
    * Click `Enable local and third party Actions for this repository`

## Deploying The Application

Let start from deploying the application to the `dev` environment. We need to make sure to use the correct image that we've built in the previous step:

* Navigate to packages page of previously forked demo-app repository: `https://github.com/<USERNAME>/demo-app/pkgs/container/demo-app`
* Copy the most recent tag image
* Configure the `dev` environment to use the image using Kustomize edit image feature. Add the following snipped to the `env/dev/kustomization.yaml` file:

```yaml
images:
- name: ghcr.io/argocon2022-workshop/demo-app
  newName: ghcr.io/<USERNAME>/demo-app
  newTag: <TAG>
```

* Commit the changes and push them to the repository
* We are ready! Navigate to the https://argocon.cd.akuity.cloud/ and create Argo CD application using the following parameters:
    * Application name: `<USERNAME>-dev`
    * Project: `<USERNAME>`
    * Repository URL: `https://github.com/<USERNAME>/demo-app-deploy`
    * Path: `env/dev`
    * Destination Cluster: `http://cluster-argocon:8001`
    * Destination Namespace: `service-<USERNAME>-dev`
    * Sync Policy: `Automated`
    * Click sync button and enjoy the demo application! (check the pod logs!)

Create the stage and prod applications using appropriate directory paths.

## Automating Dev Environment Upgrade using CI

We've not heard from any engineers that they like to manually update the image tag in the Kustomize file. Let's automate this process using Github Actions.
Jump to [Automating Image Updates](https://github.com/argocon2022-workshop/demo-app#automating-image-updates) paragraph in the [demo-app](https://github.com/argocon2022-workshop/demo-app) README.md to continue.

## Automating Staging and Production Environments

Automated promotion of Staging and Production environments requires more careful approach. The exact process is highly
opinionaed and depends on the organization. At the very least we need an approval step before promoting the
change to production. GitOps methodology and flexibility of GitHub actions can help us to achieve this.
The workflow below implementes required functionality. Please save the snippet below as `.github/workflows/promotion.yaml`, commit and push the changes to the repository. While workflow is running readon to learn more about the
workflow logic.

```yaml
name: Promote Image Change
on:
  push:
    branches:
      - main

jobs:

  promote-image-change:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
        with:
          token: ${{ secrets.DEPLOY_PAT }}
          fetch-depth: 0
      - name: Fetch Metadata
        run: git fetch origin "refs/notes/*:refs/notes/*"
      - name: Get Commit Metadata
        id: commit-metadata
        run: |
          pat='image: (.*)'; [[ "$(git notes show)" =~ $pat ]] && echo "::set-output name=IMAGE::${BASH_REMATCH[1]}" || echo ''
          pat='env: (.*)'; [[ "$(git notes show)" =~ $pat ]] && echo "::set-output name=ENV::${BASH_REMATCH[1]}" || echo ''
      - uses: fregante/setup-git-user@v1
        if: ${{ steps.commit-metadata.outputs.IMAGE }}
      - name: Promote Image Change
        if: ${{ steps.commit-metadata.outputs.IMAGE }}
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          IMAGE=${{ steps.commit-metadata.outputs.IMAGE }}
          ENV="${{ steps.commit-metadata.outputs.ENV || 'dev' }}"
          if [ $ENV == "dev" ]; then
            echo "Promoting $IMAGE to staging"
            cd env/stage && kustomize edit set image ghcr.io/argocon2022-workshop/demo-app=$IMAGE
            git add .
            git commit -m "Promote stage to $IMAGE"
            git notes append -m "image: $IMAGE"
            git notes append -m "env: stage"
            git push origin "refs/notes/*" --force && git push origin main
          elif [ $ENV == "stage" ]; then
            echo "Promoting $IMAGE to production"
            git checkout -b auto-promotion
            cd env/prod && kustomize edit set image ghcr.io/argocon2022-workshop/demo-app=$IMAGE
            git add .
            git commit -m "Promote prod to $IMAGE"
            git push origin auto-promotion --force
            gh pr create --title "Promote prod to $IMAGE" --body "Promote prod to $IMAGE" --base main --head auto-promotion
          fi
```

Engineers might make arbitrary changes in the deployment repository, but we need to automate only the image change.
To distinguish between the image change and other changes we use [git notes](https://git-scm.com/docs/git-notes) to explicitly
annotate the image promition commit. The workflow checks the commit note and if it contains the note that matches the` image: <image>`
pattern, then propagate the image change to staging and production environment.

Don't forget about the approval step requirement. The approval step is implemented using GitHub Pull Request. Instead of updating production environment
directly, workflow pushes the changes to the `auto-promotion` branch and creates a pull request, so to approve the change approver just needs to merge the pull request.

## Rendered Manifests

Using the config management tools like [Kustomize](https://kustomize.io) makes it much easier to maintain the environment-specific configuration however it also makes it much more difficult to
track changes in the Git history. That is a critical requirement for the production environments. Luckily there is a pattern called 'rendered manifests' that allows getting the best of both worlds.
The idea is to create a branch that holds plain YAML manifests which are generated by the config management tool. The process of maintaining rendered manifests branches, of course, should be automated.
A continuous Integration system like Github Actions is a perfect tool for this task.

Create the following Github Actions workflow file in the `.github/workflows` directory:

```yaml
name: Render Manifests
on:
  push:
    branches:
      - main

jobs:
  render-manifests:
    name: Render manifests
    runs-on: ubuntu-latest
    strategy:
      matrix:
        env: [dev, stage, prod]

    steps:
      - uses: actions/checkout@v3
        with:
          # fetch-depth: 0 needed to get all branches
          fetch-depth: 0
      - uses: fregante/setup-git-user@v1
      - name: Render manifests
        run: |
          kustomize build env/${{ matrix.env }} > /tmp/all.yaml
          if ! git checkout env/${{ matrix.env }} -- ; then
            git checkout --orphan env/${{ matrix.env }}
            git rm -rf .
            cp /tmp/all.yaml .
            git add .
            git commit -m "${{ github.event.head_commit.message }}"
            git push --set-upstream origin env/${{ matrix.env }}
          else
            cp /tmp/all.yaml .
          fi 
      - name: Deploy to ${{ matrix.env }}
        run: |
          if ! git diff --quiet HEAD ; then 
            git add .
            git commit -m "${{ github.event.head_commit.message }}"
            git push --set-upstream origin env/${{ matrix.env }}
          fi
```

Once the workflow is created, commit and push the changes to the repository. The workflow will create a branch for each environment and push the rendered manifests to the branch.
Next we just need to switch applications to use the rendered manifests from the branches. Lets do it!

### Summary

Let's summarize what we've built so far. We've got a multi-tenant Argo CD instance. The instance is managed using a GitOps-based process
where engineers can self-onboard their team by creating a pull request. Each application development team independently manages
its application by maintaining a separate deployment repository. Deployment manifests are generated using Kustomize and allow
engineers to avoid duplications and easily introduce environment-specific changes efficiently.

The everyday image tag changes are fully automated using the CI pipelines. On top of this, teams leverage the "rendered manifests" pattern
that turns to get Git history into a traceable audit log of every infrastructure change. Believe it or not, we've achieved way more than
many companies in several years. Next, we can focus on improving the developer's quality of life and provide features like notifications
and enable secret management.

## Notifications 

Notifications can be self-implemented by Argo CD users, for example, using Argo CD sync hooks. The sync hook is a pod that runs before or after
the application is synced, so it might be used to send a notification. However, this requires each and every engineer to find answers to questions
like:

* How to get the notification service credentials?
* Where to store the credentials?
* What should be notifiction message?
* Where to send it?

Most of the answers are the same for every team except maybe notification destinations. That is why it is better to leverage Argo CD notifications
controller that allows to configure notifications in a centralized way and enable end users to simplify specify where and when to send notifications.

First of all let's configure integration with some notification service. I'm going to use email notification for sake of simplicity.
Notifications functionality supports integration with various notification services like Slack, PagerDuty, OpsGenie and more. You can find
the notifications documentation under [this link](https://argo-cd.readthedocs.io/en/stable/operator-manual/notifications/).

After configuring integration with GMail as administrator we would have to configure triggers and templates.

You can think about triggers as a function that continuously checks the Argo CD application state and returns true if the notification should be
sent. The notification is sent when trigger's return value changes from false to true so that user get notified about an event only once when
it happens. The triggers should be configured in `argocd-notifications-cm` ConfigMap.

I want to configure two most commonly used triggers when application sync completed and failed:

```yaml
trigger.on-sync-succeeded: |
  - description: Application syncing has succeeded
    send:
    - app-sync-succeeded
    when: app.status.operationState.phase in ['Succeeded']
```

In the triggers configuration you might notice the `send` field. This field specifies the notification template that should be used to
generate the notification content. Lets defined templates as well:


```yaml
template.app-sync-succeeded: >
  email:
    subject: Application {{.app.metadata.name}} has been successfully synced.
  message: |
    {{if eq .serviceType "slack"}}:white_check_mark:{{end}} Application {{.app.metadata.name}} has been successfully synced at {{.app.status.operationState.finishedAt}}.
    Sync operation details are available at: {{.context.argocdUrl}}/applications/{{.app.metadata.name}}?operation=true .
```

Templates are using [Go templates](https://golang.org/pkg/text/template/) syntax. The template configuration includes the `message`
field that is common for all notification services. You might configure service-specific fields like `email.subject` or `slack.attachements`
for customize notifications for each notification service.

Once templates and notifications are configured, the end users need to create a subscription. To subscribe to notifications, add the following
annotation to the application definition:


```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  annotations:
    notifications.argoproj.io/subscribe.on-sync-succeeded.email: YOUR_EMAIL
```

Let's go ahead and sync the application. You should receive an email notification about the sync completion.

Next, let's configure a bit more interesting trigger, `on-deployed`. The trigger should send a notification when the new application version has been
successfully deployed. This is more difficult because application health changes frequently. The application health state might change during scaling
event and we don't want to spam users in such cases. This can be achieved using `oncePer` setting of a trigger definition. The  'oncePer' property ensures
that notification is sent only once per specified field value.

```yaml
  trigger.on-deployed: |
    when: app.status.operationState.phase in ['Succeeded'] and app.status.health.status == 'Healthy'
    oncePer: app.status.sync.revision
    send: [app-sync-succeeded]
```


## Secrets

Secrets management is a another task that could be solve by application development teams independently. However, in
this case every team would have to go through the same journey and it is better to provide a centralized solution.
More importantly the solution must not just work but also be secure and future proof. Let's consider the available options
and try using the recommended approach.

* Sealed Secrets
* Argo CD Vault Plugin
* SOPS (Secrets OPerationS)
* Vault Agent
* Secrets Store CSI Driver
* External Secrets

We've worked with all of the above solutions and found that the best fit for us and our customers is External Secrets. Let's go ahead and try
using it. The demo cluster has already been configured with External Secrets controller and fake secret storage. Store the following snipped into the `base/secret.yaml` file and don't forget to replace the referenced remote secret key:

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: my-secret
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: fake
    kind: ClusterSecretStore
  target:
    name: my-secret
  data:
  - secretKey: secret-key
    remoteRef:
      key: /<REPLACE> # replace with your secret key. Choose one of /value1 ~ /value10
      version: v1
```

We've covered main topics that address the common application developer teams challenges. Next let's talk about cluster administrators. Move to control plane repository to [continue](https://github.com/argocon2022-workshop/control-plane#automating-cluster-management).