---
# try also 'default' to start simple
theme: seriph
# random image from a curated Unsplash collection by Anthony
# like them? see https://unsplash.com/collections/94734566/slidev
background: https://source.unsplash.com/collection/94734566/1920x1080
# apply any windi css classes to the current slide
class: 'text-center'
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

# Configuration data

Find all data in this link:

https://labur.eus/deploy-volto

<svg xmlns="http://www.w3.org/2000/svg" version="1.1" viewBox="0 0 37 37" stroke="none" class="h-sm">
	<rect width="100%" height="100%" fill="#FFFFFF"/>
	<path d="M4,4h1v1h-1z M5,4h1v1h-1z M6,4h1v1h-1z M7,4h1v1h-1z M8,4h1v1h-1z M9,4h1v1h-1z M10,4h1v1h-1z M12,4h1v1h-1z M13,4h1v1h-1z M16,4h1v1h-1z M18,4h1v1h-1z M19,4h1v1h-1z M20,4h1v1h-1z M22,4h1v1h-1z M23,4h1v1h-1z M26,4h1v1h-1z M27,4h1v1h-1z M28,4h1v1h-1z M29,4h1v1h-1z M30,4h1v1h-1z M31,4h1v1h-1z M32,4h1v1h-1z M4,5h1v1h-1z M10,5h1v1h-1z M12,5h1v1h-1z M14,5h1v1h-1z M15,5h1v1h-1z M16,5h1v1h-1z M20,5h1v1h-1z M23,5h1v1h-1z M24,5h1v1h-1z M26,5h1v1h-1z M32,5h1v1h-1z M4,6h1v1h-1z M6,6h1v1h-1z M7,6h1v1h-1z M8,6h1v1h-1z M10,6h1v1h-1z M12,6h1v1h-1z M17,6h1v1h-1z M19,6h1v1h-1z M22,6h1v1h-1z M23,6h1v1h-1z M26,6h1v1h-1z M28,6h1v1h-1z M29,6h1v1h-1z M30,6h1v1h-1z M32,6h1v1h-1z M4,7h1v1h-1z M6,7h1v1h-1z M7,7h1v1h-1z M8,7h1v1h-1z M10,7h1v1h-1z M12,7h1v1h-1z M14,7h1v1h-1z M15,7h1v1h-1z M17,7h1v1h-1z M18,7h1v1h-1z M19,7h1v1h-1z M20,7h1v1h-1z M22,7h1v1h-1z M26,7h1v1h-1z M28,7h1v1h-1z M29,7h1v1h-1z M30,7h1v1h-1z M32,7h1v1h-1z M4,8h1v1h-1z M6,8h1v1h-1z M7,8h1v1h-1z M8,8h1v1h-1z M10,8h1v1h-1z M12,8h1v1h-1z M13,8h1v1h-1z M16,8h1v1h-1z M21,8h1v1h-1z M23,8h1v1h-1z M26,8h1v1h-1z M28,8h1v1h-1z M29,8h1v1h-1z M30,8h1v1h-1z M32,8h1v1h-1z M4,9h1v1h-1z M10,9h1v1h-1z M17,9h1v1h-1z M21,9h1v1h-1z M22,9h1v1h-1z M23,9h1v1h-1z M24,9h1v1h-1z M26,9h1v1h-1z M32,9h1v1h-1z M4,10h1v1h-1z M5,10h1v1h-1z M6,10h1v1h-1z M7,10h1v1h-1z M8,10h1v1h-1z M9,10h1v1h-1z M10,10h1v1h-1z M12,10h1v1h-1z M14,10h1v1h-1z M16,10h1v1h-1z M18,10h1v1h-1z M20,10h1v1h-1z M22,10h1v1h-1z M24,10h1v1h-1z M26,10h1v1h-1z M27,10h1v1h-1z M28,10h1v1h-1z M29,10h1v1h-1z M30,10h1v1h-1z M31,10h1v1h-1z M32,10h1v1h-1z M12,11h1v1h-1z M13,11h1v1h-1z M14,11h1v1h-1z M15,11h1v1h-1z M16,11h1v1h-1z M17,11h1v1h-1z M18,11h1v1h-1z M20,11h1v1h-1z M21,11h1v1h-1z M23,11h1v1h-1z M5,12h1v1h-1z M6,12h1v1h-1z M8,12h1v1h-1z M10,12h1v1h-1z M11,12h1v1h-1z M14,12h1v1h-1z M16,12h1v1h-1z M18,12h1v1h-1z M20,12h1v1h-1z M21,12h1v1h-1z M24,12h1v1h-1z M26,12h1v1h-1z M28,12h1v1h-1z M29,12h1v1h-1z M30,12h1v1h-1z M31,12h1v1h-1z M32,12h1v1h-1z M9,13h1v1h-1z M11,13h1v1h-1z M15,13h1v1h-1z M17,13h1v1h-1z M21,13h1v1h-1z M22,13h1v1h-1z M24,13h1v1h-1z M25,13h1v1h-1z M26,13h1v1h-1z M30,13h1v1h-1z M32,13h1v1h-1z M5,14h1v1h-1z M6,14h1v1h-1z M7,14h1v1h-1z M8,14h1v1h-1z M10,14h1v1h-1z M11,14h1v1h-1z M13,14h1v1h-1z M16,14h1v1h-1z M18,14h1v1h-1z M19,14h1v1h-1z M20,14h1v1h-1z M24,14h1v1h-1z M28,14h1v1h-1z M29,14h1v1h-1z M31,14h1v1h-1z M32,14h1v1h-1z M5,15h1v1h-1z M7,15h1v1h-1z M8,15h1v1h-1z M11,15h1v1h-1z M14,15h1v1h-1z M15,15h1v1h-1z M17,15h1v1h-1z M20,15h1v1h-1z M21,15h1v1h-1z M22,15h1v1h-1z M24,15h1v1h-1z M25,15h1v1h-1z M26,15h1v1h-1z M28,15h1v1h-1z M31,15h1v1h-1z M5,16h1v1h-1z M6,16h1v1h-1z M7,16h1v1h-1z M9,16h1v1h-1z M10,16h1v1h-1z M12,16h1v1h-1z M19,16h1v1h-1z M20,16h1v1h-1z M21,16h1v1h-1z M24,16h1v1h-1z M26,16h1v1h-1z M27,16h1v1h-1z M4,17h1v1h-1z M5,17h1v1h-1z M9,17h1v1h-1z M11,17h1v1h-1z M15,17h1v1h-1z M16,17h1v1h-1z M17,17h1v1h-1z M20,17h1v1h-1z M21,17h1v1h-1z M22,17h1v1h-1z M24,17h1v1h-1z M25,17h1v1h-1z M26,17h1v1h-1z M27,17h1v1h-1z M31,17h1v1h-1z M32,17h1v1h-1z M5,18h1v1h-1z M6,18h1v1h-1z M7,18h1v1h-1z M9,18h1v1h-1z M10,18h1v1h-1z M12,18h1v1h-1z M13,18h1v1h-1z M14,18h1v1h-1z M15,18h1v1h-1z M17,18h1v1h-1z M18,18h1v1h-1z M24,18h1v1h-1z M25,18h1v1h-1z M27,18h1v1h-1z M30,18h1v1h-1z M32,18h1v1h-1z M7,19h1v1h-1z M8,19h1v1h-1z M9,19h1v1h-1z M11,19h1v1h-1z M13,19h1v1h-1z M14,19h1v1h-1z M17,19h1v1h-1z M21,19h1v1h-1z M23,19h1v1h-1z M24,19h1v1h-1z M26,19h1v1h-1z M27,19h1v1h-1z M32,19h1v1h-1z M6,20h1v1h-1z M7,20h1v1h-1z M8,20h1v1h-1z M9,20h1v1h-1z M10,20h1v1h-1z M11,20h1v1h-1z M13,20h1v1h-1z M14,20h1v1h-1z M15,20h1v1h-1z M17,20h1v1h-1z M19,20h1v1h-1z M21,20h1v1h-1z M23,20h1v1h-1z M25,20h1v1h-1z M26,20h1v1h-1z M29,20h1v1h-1z M32,20h1v1h-1z M5,21h1v1h-1z M8,21h1v1h-1z M9,21h1v1h-1z M15,21h1v1h-1z M16,21h1v1h-1z M17,21h1v1h-1z M18,21h1v1h-1z M19,21h1v1h-1z M21,21h1v1h-1z M24,21h1v1h-1z M26,21h1v1h-1z M31,21h1v1h-1z M32,21h1v1h-1z M4,22h1v1h-1z M8,22h1v1h-1z M10,22h1v1h-1z M11,22h1v1h-1z M12,22h1v1h-1z M13,22h1v1h-1z M14,22h1v1h-1z M17,22h1v1h-1z M21,22h1v1h-1z M22,22h1v1h-1z M25,22h1v1h-1z M28,22h1v1h-1z M30,22h1v1h-1z M31,22h1v1h-1z M32,22h1v1h-1z M5,23h1v1h-1z M8,23h1v1h-1z M11,23h1v1h-1z M14,23h1v1h-1z M16,23h1v1h-1z M17,23h1v1h-1z M19,23h1v1h-1z M21,23h1v1h-1z M23,23h1v1h-1z M24,23h1v1h-1z M26,23h1v1h-1z M27,23h1v1h-1z M31,23h1v1h-1z M32,23h1v1h-1z M4,24h1v1h-1z M6,24h1v1h-1z M9,24h1v1h-1z M10,24h1v1h-1z M11,24h1v1h-1z M12,24h1v1h-1z M14,24h1v1h-1z M15,24h1v1h-1z M16,24h1v1h-1z M17,24h1v1h-1z M18,24h1v1h-1z M21,24h1v1h-1z M22,24h1v1h-1z M23,24h1v1h-1z M24,24h1v1h-1z M25,24h1v1h-1z M26,24h1v1h-1z M27,24h1v1h-1z M28,24h1v1h-1z M31,24h1v1h-1z M12,25h1v1h-1z M14,25h1v1h-1z M15,25h1v1h-1z M16,25h1v1h-1z M17,25h1v1h-1z M18,25h1v1h-1z M20,25h1v1h-1z M23,25h1v1h-1z M24,25h1v1h-1z M28,25h1v1h-1z M32,25h1v1h-1z M4,26h1v1h-1z M5,26h1v1h-1z M6,26h1v1h-1z M7,26h1v1h-1z M8,26h1v1h-1z M9,26h1v1h-1z M10,26h1v1h-1z M12,26h1v1h-1z M15,26h1v1h-1z M18,26h1v1h-1z M21,26h1v1h-1z M23,26h1v1h-1z M24,26h1v1h-1z M26,26h1v1h-1z M28,26h1v1h-1z M30,26h1v1h-1z M31,26h1v1h-1z M32,26h1v1h-1z M4,27h1v1h-1z M10,27h1v1h-1z M13,27h1v1h-1z M14,27h1v1h-1z M18,27h1v1h-1z M19,27h1v1h-1z M20,27h1v1h-1z M21,27h1v1h-1z M22,27h1v1h-1z M24,27h1v1h-1z M28,27h1v1h-1z M4,28h1v1h-1z M6,28h1v1h-1z M7,28h1v1h-1z M8,28h1v1h-1z M10,28h1v1h-1z M12,28h1v1h-1z M13,28h1v1h-1z M14,28h1v1h-1z M15,28h1v1h-1z M16,28h1v1h-1z M17,28h1v1h-1z M20,28h1v1h-1z M22,28h1v1h-1z M24,28h1v1h-1z M25,28h1v1h-1z M26,28h1v1h-1z M27,28h1v1h-1z M28,28h1v1h-1z M29,28h1v1h-1z M31,28h1v1h-1z M32,28h1v1h-1z M4,29h1v1h-1z M6,29h1v1h-1z M7,29h1v1h-1z M8,29h1v1h-1z M10,29h1v1h-1z M13,29h1v1h-1z M15,29h1v1h-1z M16,29h1v1h-1z M19,29h1v1h-1z M20,29h1v1h-1z M24,29h1v1h-1z M25,29h1v1h-1z M28,29h1v1h-1z M4,30h1v1h-1z M6,30h1v1h-1z M7,30h1v1h-1z M8,30h1v1h-1z M10,30h1v1h-1z M12,30h1v1h-1z M13,30h1v1h-1z M17,30h1v1h-1z M19,30h1v1h-1z M20,30h1v1h-1z M21,30h1v1h-1z M23,30h1v1h-1z M27,30h1v1h-1z M28,30h1v1h-1z M29,30h1v1h-1z M30,30h1v1h-1z M32,30h1v1h-1z M4,31h1v1h-1z M10,31h1v1h-1z M12,31h1v1h-1z M13,31h1v1h-1z M16,31h1v1h-1z M20,31h1v1h-1z M21,31h1v1h-1z M23,31h1v1h-1z M24,31h1v1h-1z M25,31h1v1h-1z M26,31h1v1h-1z M27,31h1v1h-1z M31,31h1v1h-1z M4,32h1v1h-1z M5,32h1v1h-1z M6,32h1v1h-1z M7,32h1v1h-1z M8,32h1v1h-1z M9,32h1v1h-1z M10,32h1v1h-1z M14,32h1v1h-1z M16,32h1v1h-1z M18,32h1v1h-1z M19,32h1v1h-1z M20,32h1v1h-1z M21,32h1v1h-1z M24,32h1v1h-1z M25,32h1v1h-1z M26,32h1v1h-1z M28,32h1v1h-1z M29,32h1v1h-1z M31,32h1v1h-1z M32,32h1v1h-1z" fill="#000000"/>
</svg>

---

<img src="/codesyntax.jpg" class="place-self-senter"/>

---
