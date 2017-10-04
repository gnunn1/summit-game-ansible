# Introduction

This is a playbook to get the Red Hat 2016 Summit game up and running within an Openshift environment. It is highly recommended that you view the demo of the game that was done at the 2016 Red Hat Summit [here](https://www.youtube.com/watch?v=ooA6FmTL4Dk) to understand what the game is about. As a summary, the game is a balloon popping game where the attendees can participate by playing the game while the demo is running.

Note that this game uses a microservice architecture and  requires quite a few pods to run, thus if you are using Minishift or the Red Hat CDK it is recommended that you allocate additional memory to the environment using ```minishift config set memory xxxx```. The demo uses approximatly 3.5 GB amount of memory and you will need some over and above that for rolling deployments, builds, etc.

### Prequisites

#### Imagestreams

Note the playbook will add various imagestreams and templates to OpenShift. However, it does not update the nodejs template and this application requires that nodejs version 4 and 6 be available. Check your imagestream and make sure the nodejs imagestream has these version tags, i.e.

```
oc get is nodejs -n openshift
```

If it doesn't, replace the imagestream with one that does:

```
oc project openshift
oc delete is nodejs
oc create -f https://raw.githubusercontent.com/luciddreamz/library/master/official/nodejs/imagestreams/nodejs-rhel7.json
```

#### Cluster Admin Required

The playbook uses an admin user to install the required templates and imagestreams. The admin user requires a username and password, ```system:admin``` is not used since this playbook must support remote installations of Openshift. You can create an admin user in the CDK or minishift by logging in with a username and password and the run the following command:

```
oc adm policy add-cluster-role-to-user cluster-admin <username>
```

#### Ansible Variables

You will need the variables in ```vars/game-config.yml``` to match the particulars of your Openshift installation.

### Installing The Game

To install the game, simply run the ```install.sh``` command which will run the ansible playbook. It will take approximately 10 to 15 minutes to install everything.

### Managing The Game

To manage the game, login into the admin application. The default token is ```CH2UsJePthRWTmLI8EY6```.

### Blue/Green Gamebus Deployment

_Note this is a work in progress_

This playbook installs the gamebus in a blue/green configuration with 100% of the traffic being sent initially to blue. When a game player uses the game application, the background color is automatically set based on the color. This is done through the COLOR environment variable. Valid colors include ```default```, ```blue```, ```green``` or ```canary```.

Note that simply updating the route to send all traffic to the opposite pod will not change the color of the background. This is because the game uses a persistent connection via a WebSocket and thus the connection will not be re-routed unless a new connection is established. You can do this by either having the users refresh the game or by shutting down all pods for the previous color.

A future version of this demo will include a pipeline to better manage this process in an automated fashion.

### Repositories

As mentioned previously, the game consists of a number of repositories in github, 21 in total in fact. This guide builds a subset of the game: the basic game plus the administration app, scoreboard, leaderboard and achievements are all function. This guide does not cover the Pipeline and CICD functionality that was demonstrated in the video. At the moment the following repositories are used:

| Repository | Description
|---|---|
|[vertx-game-server](https://github.com/gnunn1/vertx-game-server)| The game server that acts as an integration bus for the other microservices. The game, leaderboard and scoreboard all connect to this in order to communicate with the other components. As the name implies, this component uses the [Vert.X](http://vertx.io/) framework.
|[mobile-app](https://github.com/gnunn1/mobile-app)| This is the actual game, it is a game written in typescript using angular. It communicates both with it's own back-end server as well as the vertx-game-server.
|[mobile-app-admin](https://github.com/gnunn1/mobile-app-admin)| This is the tool to administer the game, without this app you cannot start the game. This component does not run in OpenShift, instead it is run locally on your laptop or in Amazon EC2/S3, it requires connectivity to the vertx-game-server.
|[achievement-server](https://github.com/burrsutter/vertx-achievement-service)| Manages the player achievements, it is a JEE application that runs on EAP (or Wildfly)
|[score-server](https://github.com/gnunn1/score-server)| Aggregates the player scores, runs on BRMS and uses rules to evaluate scoring. This was forked to fix a build issue as a dependency on hibernate-core was needed to compile the hibernate annotations.
|[leaderboard](https://github.com/gnunn1/leaderboard)| A javascript application that is run outside of OpenShift to display the leaderboard. It requires connectivity to the vertx-game-server.
|[scoreboard](https://github.com/gnunn1/scoreboard)| A javascript application that is run outside of OpenShift to display the scoreboard. It requires connectivity to the vertx-game-server.
