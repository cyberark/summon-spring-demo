# summon-spring-demo
Using Summon to inject secrets into Spring services
## Getting Started
This demo uses `docker-compose` to run the following services locally:
  - Conjur Community Edition
  - PostgreSQL as persistent storage for Conjur
  - PostgreSQL as persistent storage for demo service
  - Demo service (Spring)
  - Conjur CLI

For more information on these services, check out [docker-compose.yml](docker-compose.yml).

Secrets are injected into the application's environment in two ways:
 - in the application properties file (https://github.com/cyberark/summon-spring-demo/blob/master/src/main/resources/application.yml#L5-L6) 
 - and as environment variables accessible from code (https://github.com/cyberark/summon-spring-demo/blob/master/src/main/java/hello/controller/IndexController.java#L12-L13) 

In this way, the application developer does not have to be aware of the source of secrets. Summon supports multiple secrets providers, allowing choices of providers across environments while preserving the immutability of the application.

### Starting the Demo
The `bin/start` script will build any dependencies (within Docker), clean up and existing demo state and stand up freshly configured services.
```
$ ./bin/start
[2018-03-20T14:45:49-0400]: Pulling build image (maven:3.5.2-jdk-8)... Done.
[2018-03-20T14:45:50-0400]: Building demo application run-time environment image... Done.
[2018-03-20T14:45:51-0400]: Cleaning up any existing docker compose state... Done.
[2018-03-20T14:46:03-0400]: Starting docker compose containers... Done.
[2018-03-20T14:46:08-0400]: Waiting for Conjur to be ready... Done.
[2018-03-20T14:46:24-0400]: Loading policy files... Done.
[2018-03-20T14:46:28-0400]: Starting the demo service... Done.
```
### Using the demo
The demo application mocks a pet store service which controls an inventory of pets in a persistent database. The following routes are exposed:

---
`GET` `/pets`  
List all pets in inventory
##### Returns
`200`  
An array of pets in the response body
```
[
  {
    "name": "Scooter"
  },
  {
    "name": "Sparky"
  }
]
```
##### Example
```
curl localhost:8080/pets
```

---
`POST` `/pet`  
Add a pet to the inventory
##### Request
###### Headers
`Content-Type: application/json`
###### Body
```
{
  "name": "Scooter"
}
```
##### Returns
`201`
##### Example
```
curl -XPOST --data '{ "name": "Scooter" }' -H "Content-Type: application/json" localhost:8080/pet
```

---
`GET` `/pet/{id}`  
Retrieve information on a pet
##### Returns
`404`
`200`
```
{
  "id": 1
  "name": "Scooter"
}
```
##### Example
```
curl localhost:8080/pet/1
```
---
`DELETE` `/pet/{id}`  
Remove a pet from inventory
##### Returns
`404`
`200`
##### Example
```
curl -XDELETE localhost:8080/pet/1
```

### Stopping the Demo
The `bin/stop` script will clean up existing demo state.
```
$ ./bin/stop
[2018-03-21T10:53:19-0400]: Cleaning up any existing docker compose state... Done.
```

## Understanding the Demo
1. Conjur policy ([hello.yml](conjur/hello.yml)) is loaded as part of the onboarding of this service
2. Secret values are populated before the service is started. See [start](bin/start#L37-L40).
3. A [secrets.yml](secrets.yml) file is created to declare the secrets this application will consume, and mapping their name in Conjur to environment variables.
4. Our application configuration file is updated to read the PostgreSQL username and password from the environment. See  [application.yml](src/main/resources/application.yml#L5-L6).
5. We can also use `Value` annotations to populate variables within our code to secrets provided by `summon`. See [IndexController.java](src/main/java/hello/controller/IndexController.java#L12-L13).
6. Our Java service is run using `summon`. Summon is configured as the `ENTRYPOINT` of our [Dockerfile](Dockerfile#L18), and our run command defined within [docker-compose.yml](docker-compose.yml#L49). We end up with a final run command of `summon -f /etc/secrets.yml java -jar /target/hello-0.1.0.jar`.