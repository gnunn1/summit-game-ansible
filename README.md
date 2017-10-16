<p align="center">
  <img src="https://github.com/gnunn1/summit-game-ansible/blob/master/docs/img/game.png?raw=true" alt="Game Screenshot"/>
</p>

# Introduction

This is a playbook to install the Red Hat 2016 Summit game in an Openshift environment. It is highly recommended that you view the demo of the game that was done at the 2016 Red Hat Summit [here](https://www.youtube.com/watch?v=ooA6FmTL4Dk) to understand what the game is about. As a summary, the game is a balloon popping game where the attendees can participate by playing the game while the demo is running.

Note that this game uses a microservice architecture and  requires quite a few pods to run, thus if you are using Minishift or the Red Hat CDK it is recommended that you allocate additional memory to the environment using ```minishift config set memory xxxx```. The demo uses approximatly 4 GB of memory and you will need some over and above that for rolling deployments, builds, etc.

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

You will need to update the variables in ```vars/game-config.yml``` to match the particulars of your Openshift installation.

### Installing The Game

To install the game, simply run the ```install.sh``` command which will run the ansible playbook. It will take approximately 10 to 15 minutes to install everything.

### Managing The Game

To manage the game, login into the admin application. The default token is ```CH2UsJePthRWTmLI8EY6```. If you change the token in the game-server, make suire to update it in gamebus-pipeline as well.

### Blue/Green Gamebus Deployment

This playbook installs the gamebus in a blue/green configuration with 100% of the traffic being sent initially to blue. The blue/green gamebus environments are configured as a single vert.x cluster and it uses multicast which is only supported in OpenShift 3.5 and later only.

The background of the game client reflects the environment color the player is connected to. A ```gamebus-pipeline``` is included which will build and deploy the application as well as change the route to the opposite color. This pipeline was adapted from the coolstore-microservice demo.

Note that simply updating the route to send all traffic to the opposite pod will not change the color of the background automatically. This is because the game uses a persistent connection via a WebSocket and thus the connection will not be re-routed unless a new connection is established. The last stage of the pipeline will initiate a reconnect for game and admin clients by sending a curl request to the idle game-server, this should force everyone to the new one. However once in a while it doesn't take and some players may need to do a manual refresh of their browser so be prepared for that.

When a game player connects the game client, the background color is automatically set based on the color of the environment making it easy for the audience to understand when the environment has been flipped. This is done through the COLOR environment variable. Valid colors include ```default```, ```blue```, ```green``` or ```canary```. This also means that setting the background in the admin application is ignored when the COLOR environment variable is present.

This demo has been tested once with a group of people and it worked well, however feedback is always welcome.

### Repositories

The original game consisted of a number of repositories in github, 21 in total in fact. This version builds a subset of the game: the basic game plus the administration app, scoreboard, leaderboard and achievements are all functional. This version does not include the Pipeline and CICD visualization that was demonstrated in the video. At the moment the following repositories are used:

| Repository | Description
|---|---|
|[vertx-game-server](https://github.com/gnunn1/vertx-game-server)| The game server that acts as an integration bus for the other microservices. The game, leaderboard and scoreboard all connect to this in order to communicate with the other components. As the name implies, this component uses the [Vert.X](http://vertx.io/) framework.
|[mobile-app](https://github.com/gnunn1/mobile-app)| This is the actual game, it is a game written in typescript using angular. It communicates both with it's own back-end server as well as the vertx-game-server.
|[mobile-app-admin](https://github.com/gnunn1/mobile-app-admin)| This is the tool to administer the game, without this app you cannot start the game. This component does not run in OpenShift, instead it is run locally on your laptop or in Amazon EC2/S3, it requires connectivity to the vertx-game-server.
|[achievement-server](https://github.com/burrsutter/vertx-achievement-service)| Manages the player achievements, it is a JEE application that runs on EAP (or Wildfly)
|[score-server](https://github.com/gnunn1/score-server)| Aggregates the player scores, runs on BRMS and uses rules to evaluate scoring. This was forked to fix a build issue as a dependency on hibernate-core was needed to compile the hibernate annotations.
|[leaderboard](https://github.com/gnunn1/leaderboard)| A javascript application that is run outside of OpenShift to display the leaderboard. It requires connectivity to the vertx-game-server.
|[scoreboard](https://github.com/gnunn1/scoreboard)| A javascript application that is run outside of OpenShift to display the scoreboard. It requires connectivity to the vertx-game-server.

### Random Thoughts

I've wavered between making the blue/green instances a single vert.x cluster versus separate clusters. The single cluster is easier because the game state, players and other persistent information are carried between the two when switching from blue to green making for a more seamless transition. However it goes against the idea of each color being independent since the eventbus queues would be clustered across both. This means events that should be specific to green or blue get applied to both. For example, one issue I had to deal with is configuration messages being served from the idle environment resulting in the game-client getting the wrong color.

You can try running the clusters as separate clusters by copying the contents of ```cluster-seperate.xml``` into ```cluster.xml``` in the vertx-game-server repository. When run as separate clusters it uses the Kubernetes discovery mechanism that was implemented by the initial Summit development team rather then multicast. 

It would be nice if things like the game state, player identifiers, configuration, etc would be moved to an externalized persistent store like JDG, Hazelcast, etc. Hopefully down the road I'll have a chance to dig into this some more.

Another thing I would like to do is deploy a canary on the fly. It would be relatively trivial to create a pipeline to do this as I believe no existing code changes would be required.

### Credits

A big thank you to the folks who originally developed this code for the 2016 Red Hat Summit, they did all the hard work so a big kudos to them.