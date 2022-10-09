---
# try also 'default' to start simple
theme: seriph
# random image from a curated Unsplash collection by Anthony
# like them? see https://unsplash.com/collections/94734566/slidev
background: https://source.unsplash.com/collection/94734566/1920x1080
# apply any windi css classes to the current slide
class: "text-center"
# https://sli.dev/custom/highlighters.html
highlighter: shiki
# show line numbers in code blocks
lineNumbers: false
# some information about the slides, markdown enabled
info: |
  ## Slidev Starter Template
  Presentation slides for developers.

  Learn more at [Sli.dev](https://sli.dev)
# persist drawings in exports and build
drawings:
  persist: false
# use UnoCSS
css: unocss
---

# How to deploy Volto sites automatically in a no-docker scenarios

---

# Scenario

- Standard Plone 5.2 sites
- Volto developments
- DigitalOcean standard droplets
- Non-docker deployments

<!--
This last year we have been working on updating two sites from old Plone 4.3 based setups to Plone 5.2 and Volto.

We are not yet in the docker world, at least for Plone & Volto development and deployments except for a site we have been working at the EEA (which is already Docker based both for Plone and Volto).

In this case we didn't want to change our buildout based deployment, which is deployed to DigitalOcean droplets, so we needed a way to deploy or Volto site to such servers.

-->

---

# First approach

- Let's copy our Volto site and run `yarn build` there.

<!--

That's was our first idea, because that's what we do with buildout and Plone.

But that didn't work.

-->

---

# Running `yarn build` on the server

- The build took half an hour
- The build broke after half an hour
- We had to upgrade the server to run the build

<!--
We found a lot of issues when trying to run yarn build on the server. The first one was the time that it took to get the build. It was very long. Longer than running buildout.

Moreover we had to double the RAM and CPU capabilities of the server, because otherwise the server refused to run the build
-->

---

# Solution? Automation!

- Automate the addon package creation
- Automate the frontend package compilation
- Automate the release and publishing process

<!--

So to try to solve the problems we had building that, we tried to replicate the existing pipelines we had on the EEA project to achieve our goals:

-->

---

# Problems

- Private addons: no public npmjs registry packages
- Private npm registry?
- Is not that easy...

<!--

The main problem we had is that we were not developing our customer code on public Github repositories so we couldn't release and upload the packages we were creating to the public npm registry, so we started looking for an alternative private npm registries.

We already have a similar registry for our clients' Python code a "private pypi registry", which is just a password protected, nginx directory, so we thought that could be ease to build such a thing for npm too, but we found that is not that easy as in python.
-->

---

# Verdaccio to the rescue

- Private npm registry
- Docker based installation
- Upstream-linked to npmjs

<img src="/verdaccio.png" />

<!--

After looking for private npm registries, we found Verdaccio. It is a private node-js private proxy registry, which allows publishing packages and also acting as a mirror of the official npm registry.

It looks like in JavaScript world there is no such thing like the "find-links" we have in buildout, so all packages must be download from a single place.

Verdaccio allows installing packages from npmjs transparently: you configure yarn or npm to download packages from your Verdaccio install and it will manage to download the packages that are not directly published there from the npm registry.

-->

---

# Password protecting Verdaccio

- Configure Verdaccio to work only in authenticated mode
- Configure our npm and yarn to use authentication
- Install and setup the Verdaccio instance
- Publish the configuration on GitHub

<!--

So we only had to password protect Verdaccio following its yaml configuration and we created a token.
 But the hardest part was configuring npm and yarn to use our private registry and use authentication to install packages.

The configuration needed to deploy, install and configure Verdaccio is available in our GitHub accound, and I will share the URL of it later.

-->

---

# .npmrc

```
//code.codesyntax.com/npm/:_authToken=<REDACTED>
always-auth=true

```

# .yarnc

```
registry "https://code.codesyntax.com/npm/"

```

<!--

This is the content that we had to add in our frontend package to use our private registry to install and build the frontend.

-->

---

# Let's automate it

- Automatic releasing of addons to Verdaccio
- Automatic install/build of frontend
- Automatic deployment of frontend

<!--
So if we have a private npm registry, we can now publish our addons to it, and use specific versioned packages to build our frontends.

This could be done manually, but we thought that we could use GitLab CI-CD to do it for us.

We are already using GitLab to host our clients' code and in some cases we are using GitLab CI-CD also to run some automatic deployments of a django project.

We prepared a CI pipeline to do so
-->

---

# Git process

## Addons

- We develop in a `develop` branch
- When ready, we open a merge-request to `main`
- CI pipeline is run, and if OK, the merge-request is ready
- When merged, another pipeline is run to build and publish the package
- When finished an e-mail is sent through Mailjet API

## frontend

- We update the addon version in `develop` branch
- When ready, we open a merge-request to `main`
- CI pipeline is run, and if OK, the merge-request is ready
- When merged, another pipeline is run to build the frontend and scp to server and supervisor restart
- When finished an e-mail is sent through Mailjet API

---

# volto-addon releasing

```yaml
stages:
  - publish

publish:
  ...

  before_script:
    - ...
    - export CI_PUSH_REPO=$(echo "$CI_REPOSITORY_URL" | sed -e "s|.*@\(.*\)|git@\1|" -e "s|/|:/|" )
    - git remote set-url --push origin "ssh://${CI_PUSH_REPO}"
    - git checkout -B $CI_BUILD_REF_NAME && git pull origin $CI_BUILD_REF_NAME && git push origin -u $CI_BUILD_REF_NAME
    - 'echo "//${CODESYNTAX_NPM_REPO_URL}:_authToken=${CODESYNTAX_REPO_NPM_TOKEN}" > .npmrc'
    - 'echo "always-auth=true" >> .npmrc'
    - 'echo "registry \"https://${CODESYNTAX_NPM_REPO_URL}\"" > .yarnrc'
    - yarn global add release-it
  script:
    - release-it --ci --minor --npm.skipChecks
    - git checkout develop
    - git pull origin main
  after_script:
    - npm version prerelease --preid=dev
    - git push origin -u develop
    - git checkout main
```

<!--
This is a simplified version of our ci pipeline.

In essence it prepares the repo syncing it with the corresponding remote (this is needed because the gitlab ci is run in a detached-head mode and has no access to the remote), adding the .npmrn and .yarnrc files, and then releasing it with release-it.

We also run some steps after the release, such as marking the develop branch version with a 'dev' marker, and sending an e-mail using MailJet to our developers' mailing list.
-->

---

# frontend releasing (1/2)

```yaml
image: node:16.15.1-alpine

mergerequest:
  stage: mergerequest
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
  before_script:
    - yarn jsconfig:prod
  script:
    # - !reference [.template_get_artifacts, script]
    - yarn install --frozen-lockfile
    - yarn jsconfig:dev
    - git checkout -- .yarnrc yarn.lock
    - yarn i18n:ci
    - yarn jsconfig:prod
    - yarn build
```

<!--

This is a simplified version of the frontend release pipeline.

We basically remove all the development configuration (using the yarn jsconfig:dev we reset the jsconfig file not to use development packages) and run the build.

Previous to this step we manually modify the package.json to pin here the new version released in the previous addon releasing process.

As last step, we save the build as a Gitlab artifact and pass it to the next step.

-->

---

# frontend releasing (2/2)

```yaml
release:
  stage: release-publish
  rules:
    - if: $CI_COMMIT_REF_NAME == "main"
  script:
    - git checkout -- .yarnrc yarn.lock
    - release-it --ci --minor --npm.skipChecks

publish:
  stage: release-publish
  rules:
    - if: $CI_COMMIT_REF_NAME == "main"
  script:
    - rsync -a build/* $SSH_SERVER_USER@$SSH_SERVER_NAME:$SSH_SERVER_BUILDOUT_PATH/frontend/build/
    - rsync -a node_modules $SSH_SERVER_USER@$SSH_SERVER_NAME:$SSH_SERVER_BUILDOUT_PATH/frontend/
    - ssh -T $SSH_SERVER_USER@$SSH_SERVER_NAME "$SSH_SERVER_BUILDOUT_PATH/bin/supervisorctl restart $VOLTO_SUPERVISOR_NAME"
```

<!--

This is the second step we reset the artifact created in the previous one, we create a release (it just creates a tag and release in GitLab), and then scp-s the build and node_modules to the server and restart the supervisor.


-->

---

# GitLab CI configuration is hard

- Stages, steps, artifacts
- Artifacts are shared between jobs but not between pipelines
- Use artifacts to cache already built things
- Use variables for everything

<!--

You need to carefully check the Gitlab CI configuration because it has some problems you may face when configuring it, specially when creating and sharing artifacts between jobs.

You want to share the builds between jobs not to run yarn build more than one time, and thus save some time in your pipeline.

We run the build first when doing a merge-request to check that the build is OK, and then again when merging it to really create the files to be deployed.

We have also a large set of variables that can be configured at organization level in GitLab and can be used in package level.

This way, we set the required SSH keys or the keys needed to send emails through Mailjet in the organization level and leave the project level variables in the frontend or addon packages.

-->

# How to do configuration updates?

- We share configuration in a GitHub repo
- Python script to download and update config files

```
python update-templates.py -frontend
python update-templates.py -theme
python update-templates.py -myself

```

<!--

We decided to centralize the configuration scripts in a GitHub repo and prepare a script to update the files in any project.

This way we copy the update-templates.py script to our addon and frontends, and can run the script to update the configuration to our standards
-->

---

# Configuration

- Verdaccio

---
