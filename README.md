## Spring Cloud Fortune Teller
###Microservices lab with Spring Cloud Services in Pivotal Cloud Foundry

Steps 1-3 in this lab are required to get the project running properly in PCF. Steps 4-6 are optional, however illustrate how to make use of the Spring Cloud Service features. All the POM dependencies are already included for each step. You wimple clone this repo and follow the instructions to configure and push into PCF.

####1. *Set up Config Server Repo*

This step allows you to adjust logging and security without redeploying your app.
* Clone App config repo:   https://github.com/bbertka/fortune-teller-app-config
* Using the marketplace, create a Config Server service instance, but do not bind to an app. Set GIT URI to:  https://github.com/bbertka/fortune-teller-app-config.git

Notice in the repo that each app has its own YML file with specific configurations.

####2. *Set up Fortune Service* 

This service will serve fortunes to the greeting service, it will pull its initial logging and Eureka registration methods from the Config Server repo.
* cd fortune-service
* mvn clean package
* cf push fortune-service -p target/fortune-service-0.0.1-SNAPSHOT.jar -m 512M --random-route --no-start
* cf create-service p-service-registry standard service-registry
* cf bind-service fortune-service config-server
* cf bind-service fortune-service service-registry
* cf set-env fortune-service CF_TARGET https://api.your.domain.com
* cf start fortune-service

Its mandatory that CF_TARGET is set so the service discovery works properly.

####3. *Set up Greeting Service* 

This service will provide a front end UI to the system, like fortune-service, it will pull its initial logging and Eureka registration method from the Config Server repo.
* cd greeting-service/
* mvn clean package
* cf push greeting-service -p target/greeting-service-0.0.1-SNAPSHOT.jar -m 512M -i 2 --random-route --no-start
* cf bind-service greeting-service config-server
* cf bind-service greeting-service service-registry
* cf set-env greeting-service CF_TARGET https://api.your.domain.com
* cf start greeting-service

By now you should be able to see each app registered within Service Discovery Management console.  Note how each app registers differently, with either a Route, or a container IP address.  When we apply client side load balancing with Ribbon, each registration method should be direct, otherwise we are adding reduntant load balancing via the Go router which defeats the purpose of using Ribbon in the first place.

####4. *Change Eureka Restration Method*

We want for demo purposes to have both services register directly with IP addresses.
* cd fortune-teller-app-config
* Set "registrationMethod: direct" in fortune-service.yml
* git push the changes
* cf restart fortune-service

Now both the services are registering _directly_ with their IP addresses.  Why did we have to restart? Config server doesn't support auto refresh on all parameters, and registrationMethod is merley one.

####5. *Change Logging Levels to DEBUG using Config Server* 

We will see that multiple instances of apps are running in a bounded context, hence refreshing an app, really means refreshing the instance we have connected to.
* Open two terminal windows:
	* cf logs greeting-service | grep DEBUG
	* cf logs greeting-service | grep INFO
* Open a third terminal window
	* Set "logging: DEBUG" in greeting-service.yml
	* git push the changes
	* curl -X POST https://greeting-service-route/refresh

Notice that only 1 instance is in DEBUG logging mode -- so we need cloud bus refresh to hit them all

####6. *Set up Cloud Bus*

To get around the bounded context of a microservice, we add a Cloud Bus that is able to refresh all the instances for us with one command sent to a single instance
* cf create-service p-rabbitmq standard cloud-bus
* cf bind-service greeting-service cloud-bus
* cf push greeting-service -p target/greeting-service-0.0.1-SNAPSHOT.jar -m 512M -i 2
	
####7. *Change Logging Levels back to INFO for all instances*

Now that we have a Bus set up, we can adjust logging for all our instances at once
* Set "logging: INFO" in greeting-service.yml
* git push the changes
* curl -X POST https://greeting-service-route/bus/refresh

####Coming Soon
* Ribbon Client Side Load Balancing
* Circuit Breaker Pattern  & Turbine Dashboard
