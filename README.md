Set up Config Server Repo:
	Clone App config repo:   https://github.com/bbertka/fortune-teller-app-config
	Create Config Server service instance, when ready set GIT URI to:  https://github.com/bbertka/fortune-teller-app-config.git

Set up Fortune Service:
	cd fortune-service  
	mvn clean package
	cf push fortune-service -p target/fortune-service-0.0.1-SNAPSHOT.jar -m 512M --random-route --no-start
	cf create-service p-service-registry standard service-registry
	cf bind-service fortune-service config-server
	cf bind-service fortune-service service-registry
	cf set-env fortune-service CF_TARGET https://api.vert.fe.gopivotal.com
	cf start fortune-service

Set up Greeting Service:
	cd greeting-service/
	mvn clean package
	cf push greeting-service -p target/greeting-service-0.0.1-SNAPSHOT.jar -m 512M -i 2 --random-route --no-start
	cf bind-service greeting-service config-server
 	cf bind-service greeting-service service-registry
 	cf set-env greeting-service CF_TARGET https://api.vert.fe.gopivotal.com
 	cf start greeting-service

Change Eureka Restration Method:
	cd fortune-teller-app-config
      	Set "registrationMethod: direct" in greeting-service.yml
	git push the changes
	cf restart greeting-service

Change Logging Levels to DEBUG for single instance:
	Open two terminal windows:
		cf logs greeting-service | grep DEBUG
		cf logs greeting-service | grep INFO
        Set "logging: DEBUG" in greeting-service.yml
        git push the changes
	curl -X POST https://<app route>/refresh
	Notice that only 1 instance is in DEBUG logging mode -- so we need cloud bus refresh to hit them all

Set up Cloud Bus (when push, the config is set across all instances)
        cf create-service p-rabbitmq standard cloud-bus
        cf bind-service greeting-service cloud-bus
        cf push greeting-service -p target/greeting-service-0.0.1-SNAPSHOT.jar -m 512M -i 2
	
Change Logging Levels back to INFO for all instances without Restart:
        Set "logging: INFO" in greeting-service.yml
        git push the changes
        curl -X POST https://<app route>/bus/refresh
