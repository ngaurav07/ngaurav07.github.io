---
title: 'Players Matchmaking Using Open Match, Agones and Minikube'
date: 2023-01-02
permalink: /posts/players-matchmaking-in-kubernetes/
tags:
  - kubernetes
  - golang
  - agones
  - architecture
---
Matchmaking is the process of connecting players together for online play sessions in multiplayer video games. Open Match is a framework that simplifies building players' matchmakers, which runs on Kubernetes.

### A Story About Matchmaking
A few years ago, I played games and had a bad experience when I matched with players with higher or lower levels. When continuously matching with higher-level players, it was difficult for me to win the game. As well as, when matching with players of lower levels, it's easy to win against them.

Both these cases led me to uninstall the game, and the sole reason behind this was my bad user experience.

### Things to Know for Great Matchmaking
1. Players must be matched with the other players having a low difference in their XPS or levels.
2. For low latency, players residing in the same country or region must be matched, and the server must be as close as possible.
3. Players with the same coins and the same playing regions must be matched.
4. Players can make a private lobby or room as per their requirements.
5. In the case of a few players, the matchmaker can forget the above rules and match players. For example, if there are only two players where one player has XP 5 and another has XP 50, then they must be matched, keeping their XP information aside. This also applies to the coins and latency if the number of players is low.
6. In case of necessity, matchmaking must also create the game server and forward this information to the players.
7. It must efficiently distribute the players. For example, let's assume there are 5 players, the maximum number of players is 4, and the minimum number of players is 2. Instead of distributing 4 people in one game server and making one extra player wait for others, matchmaking must distribute 3 players in one game server and 2 in another.

### About Open Match
Let's discuss a little about the matchmaking framework open-sourced by Google.

This framework is written in go and runs in Kubernetes. Some features of the Open Match are as follows:

1. Can scale up and down (horizontal scaling) as per the requirement being running in Kubernetes
2. Provides an API for accessing its endpoint – SwaggerUI for APIs
3. Has Backfill services for replacing the missing player with someone new
4. Uses Telemetry services like Grafana, Prometheus, Jaeger, and Stakdriver
5. Has TLS Encryption
6. Uses Redis as a database for saving tickets.

### Open Match Installation in minikube
As an Open Match runs inside Kubernetes, you must install Kubernetes on your machine. Please refer to our other blog post, Kubernetes Deployments with Helm detailing it.

### Step 1: Set Up a minikube Cluster
```bash
minikube start --cpus=3 --memory=2500mb
```
An Open Match can be installed through YAML as well as Helm.

### Step 2: Install Open Match
We are going to install Open Match through YAML. We can customize the YAML file after downloading it, but we can start directly now.
```bash
kubectl apply --namespace open-match \
-f https://open-match.dev/install/v1.4.0-rc.1/yaml/01-open-match-core.yaml
```
After successfully running the above code, let's see the Open Match pods.

```bash
kubectl get pods -n open-match
```
```bash
Output:

NAME                                READY   STATUS              RESTARTS   AGE
om-backend-76d8d76c96-fmhmn         0/1     ContainerCreating   0          3m53s
om-frontend-57fc9f6b66-86hxj        0/1     ContainerCreating   0          3m53s
om-query-799d8549d4-5qpgx           0/1     ContainerCreating   0          3m53s
om-swaggerui-867d79b885-m9q6x       0/1     ContainerCreating   0          3m54s
om-synchronizer-7f48f84dfd-j8swx    0/1     ContainerCreating   0          3m54s
```
> **Note:** ✍️ Open Match needs to be customized to run as a Matchmaker. This custom configuration is provided to the Open Match components via a ConfigMap (om-configmap-override). Thus, starting the core service pods will remain in ContainerCreating until this config map is available.

Run the command below to install the default evaluator in the Open Match namespace and configure Open Match to use it.
```bash
kubectl apply --namespace open-match \

-f https://open-match.dev/install/v1.4.0-rc.1/yaml/06-open-match-override-configmap.yaml

-f https://open-match.dev/install/v1.4.0-rc.1/yaml/07-open-match-default-evaluator.yaml
```
When Open Match is fully installed, the following output can be seen:

```bash
kubectl get pods -n open-match
```
The output must look like this:
```bash
NAME                                       READY   STATUS    RESTARTS   AGE
open-match-backend-678dfb8464-bh9zp        1/1     Running   0          100m
open-match-evaluator-6db6468d56-2ft24      1/1     Running   0          100m
open-match-frontend-584c576f4b-shxvx       1/1     Running   0          100m
open-match-query-66bcfbf744-6kxcx          1/1     Running   0          100m
open-match-redis-node-0                    3/3     Running   0          100m
open-match-redis-node-1                    3/3     Running   0          99m
open-match-swaggerui-7476b64b94-68tbn      1/1     Running   0          100m
open-match-synchronizer-5b8948dd46-5b4f6   1/1     Running   0          100m
```
*Output after Open Match Installation*
To successfully integrate Open Match in our games, we need to customize a few things or write the codes for these services. We can write the code in any language we wish by connecting to the GRPC or REST endpoints.

### Open Match APIS
We must access these hostnames and endpoints to create the matchmaker's main logic.
```bash
swaggerui:
hostName: om-swaggerui
httpPort: 51500

query:
hostName: om-query
grpcPort: 50503
httpPort: 51503

frontend:
hostName: om-frontend
grpcPort: 50504
httpPort: 51504

backend:
hostName: om-backend
grpcPort: 50505
httpPort: 51505
```
### Components to Know
- **Ticket**: A basic matchmaking entity in Open Match that represents a player (or group of players) requesting a match.
- **Assignment**: A mapping from a game server assignment to a Ticket.
- **Match**: A collection of tickets and the metadata of a match.
- **Match Function(MMF)**: The core matchmaking logic is implemented as a service invoked by Open Match to generate matches. It takes lists of tickets (which meet certain constraints) as input and returns any number of matches. The Match() function simply pairs any two players into a match for this basic demo.
- **Director**: A component that requests Open Match for matches and then sets assignments on the tickets in the matches found.
- **Client**: A service that translates in-queued players into tickets in Open Match understandable language; instructs Open Match that a player is currently looking for a match.
- **Profile**: Conditions denoting what type of players to match. For example, match players with a 5 XP difference.
![Agones Flow chart](https://open-match.dev/site/images/demo-match-sequence.png)
                *Image used from Official Open Match site*

### Flow for the Matchmaking
1. Players connect to the frontend (client) and send their information like the XP, coin and region, then wait for the assignment.
2. The frontend creates the ticket and sends it to the Open Match's frontend.
3. The Director creates the profiles and connects them to the Match() function.
4. The **Match()** function takes the profiles, also called pools, from the Director and extracts the tickets from the Open Match query. Loops in the given profile create the matches (keeping the players in the same array) and return the matches to the Director.
5. The Director gets the matches, creates an assignment, connects the Open Match backend and returns it to the Open Match backend. The Director also needs to create the game server, Agones. Inside the assignment, they send the players the game server information like IP and port.
6. Frontend fetches the assignment from the Open Match frontend and sends it back to the player.

### Conclusion
In this blog, we learned about the importance of matchmaking in online games, using an open-source framework by Google, the installation of Open Match in Kubernetes, and the aspects of Open Match.

In the upcoming post, we will learn about connecting Agones and Open Match for allocating game servers through Open Match. We will be using Open Match in cloud clusters like digital ocean, telemetry services provided by Open Match, and so on.