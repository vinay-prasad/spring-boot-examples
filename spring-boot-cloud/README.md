# Spring Boot Cloud example

## Netflix Eureka : Service discovery and registry
	Application related properties are usually defined in either individual properties file or hardcoded directly into the code. 
	If a change to such properties lile listening port no is required then it becomes very cumbersome job to adapt the changes. 
	Specially, if there are multiple applications which is very common in microservice type architecture
 
 	With Netflix Eureka server, hardcoding can be removed (e.g port details in producer-consumer example)
 	If a service is changed like its port or location, dependent services are not impacted if services are using Eureka service 
	discovery and registry
 
	Implementation of Eureka  Service discovery and registry
 	1. Develop two services producer and consumer (Rest Application)
 	2. Implement service registration for Producer application
 	3. Implement service discovery for Consumer application
 
	Eureka server implmentation
 	Dependency management configuration in pom for spring-cloud-dependencies. The reason is to effectively adapt to the latest 
	changes/releases made by Netflix team
 
	Updates to Producer (Service Registry)
 	1. add dependency spring-cloud-starter-eureka and dependency management spring-cloud-dependencies in the pom
 	2. add @EnableDiscoveryClient in the SpringBootApplication
 	3. add application.property to refer the eureka server running on 8090 
	"eureka.client.serviceUrl.defaultZone=http://localhost:8090/eureka"
 	4. Restart the producer application. An "UNKNOWN" application will be registered now at Eureka server 
	"http://localhost:8090/"
 	5. Create a bootstrap.properties to provice a unique name to this application and restart it 
	"spring.application.name=employee-producer"
 	6. Restart the producer application. An "employee-producer" application will be registered now at Eureka server 
	"http://localhost:8090/"

	Updates to Consumer (Service Discovery) 
 	1. add dependency spring-cloud-starter-eureka and dependency management spring-cloud-dependencies in the pom
 	2. add @EnableDiscoveryClient in the SpringBootApplication for its own discovery however its not required for dicovering 
	producer
 	3. add application.property to refer the eureka server running on 8090 
	"eureka.client.serviceUrl.defaultZone=http://localhost:8090/eureka"
 	4. Create a bootstrap.properties to provice a unique name to this application and restart it 
	"spring.application.name=employee-consumer"
 	5. Add below code to find the URI of producer using DiscoveryClient and use the URL to invoke producer
	
	 	@Autowired
		DiscoveryClient discoveryClient;
		public void getEmployee() throws RestClientException, IOException {
		
			List<ServiceInstance> instances=discoveryClient.getInstances("employee-producer");
			ServiceInstance serviceInstance=instances.get(0);			
			String baseUrl=serviceInstance.getUri().toString() + "/employee";
			
 	6. Restart the producer application. An "employee-consumer" application will be registered now at Eureka server 
	"http://localhost:8090/" and this should be able to discover producer via the unique id
 
## Netflix Ribbon + Eureka : load balancing and Service discovery and registry

	Used to obtain Load balancing at client level without depending on dedicated HW like F5 load balancers

	Changes to employee-producer (Which needs to be highly available)
 	1. Different port numbers for each producer and "eureka.instance.instanceId" to get unique application name in Eureka server
	server.port=8093
	eureka.client.serviceUrl.defaultZone=http://localhost:8090/eureka
	eureka.instance.instanceId=${spring.application.name}:${random.value}
 	2. Run the two applications in parallel and verify multiple producers in Eureka server
 
	Changes to employee-consumer (Which is going to invoker producers without knowing a specific producer)
 	1. Add dependency "spring-cloud-starter-ribbon"  
 	2. Autowire LoadBalancerClient annotation in the consumer (Earlier it was DiscoveryClient)
 	3. Update the controller code to get the URI and restart the consumer		
 
		@Autowired
		private LoadBalancerClient loadBalancer;
		public void getEmployee() throws RestClientException, IOException {
			ServiceInstance serviceInstance=loadBalancer.choose("employee-producer");
			System.out.println(serviceInstance.getUri());
			String baseUrl=serviceInstance.getUri().toString() + "/employee"; 
 
	Consumer picks any available producer for making request.


## Netflix Hysterix : FallBack and CircuitBreaker
	Hystrix is a latency and fault tolerance library designed to 
	- isolate points of access to remote systems,services and 3rd party libraries, 
	- stop cascading failure and enable resilience in complex distributed systems where failure is inevitable 
 	
	Two features of Hystrix-
 	1. Fallback method (Like a try catch, where on an exception a controlled operation is performed)
 	2. Circuit Breaker
 
 ### Fallback method
     To implement Fallback method, the producer needs to be updated with few simple changes
     1. add hystrix depedency in pom "spring-cloud-starter-hystrix"
     2. add annotation to SpringBootApplication @EnableCircuitBreaker
     3. update the controller to decalare and define Fallback method
     
     	@RequestMapping(value = "/employee", method = RequestMethod.GET)
		@HystrixCommand(fallbackMethod = "getDataFallBack")
		public Employee firstPage() {
		// actual method
		}
		
		public Employee getDataFallBack() {
		// fallback method
		}
		
## Spring cloud config : Centrally manage all the common/global/shared config
	Our producer consumer example has common configs which can be shared globally. 
	Spring cloud config provides an efficient way to achieve this
	
	To implement this it requires to create a new service employee-config-server and 
	few updates in the producer-consumer application.properties and pom files
	
	Changes to the employee-config-server
	1. add dependency spring-cloud-config-server
	2. create common application.properties under src/main/resources/common-config to hold all common properties e.g.
		eureka.client.serviceUrl.defaultZone=http://localhost:8090/eureka
	3. update the actual application.properties to use native/git based config
		spring.profiles.active=native
		server.port=8888
		spring.cloud.config.server.native.search-locations=classpath:/common-config
	4. update pom to enable filtering to read src/main/resources/common-config/application.properties
	5. enable config server by using annotation @EnableConfigServer
	
	Changes to the producer-consumer
	1. add dependency spring-cloud-starter-config and remove/update application.property for 
		#eureka.client.serviceUrl.defaultZone=http://localhost:8090/eureka
	
	For git based cloud config, path of repo needs to be mentioned in config-server's application properties
	
## Spring zuul : Provides a firewall/proxy/gateway type of architecture
	Zuul acts as front door for all the requests and routes is to backend systems.
	
	1.	employee producer
	2.	Eureka Server
	3.	employee-zuul-service
	4.	employee consumer-zuul
		
	implement zuul server
	-	add dependency spring-cloud-starter-zuul
	-	add classes for 4 types of Zuul supported filters Pre, Post, Route, Error
	-	add annotation @EnableZuulProxy to the sprinboot application class
	-	add below application and bootstrap properties
		
		
		zuul.routes.producer.url=http://localhost:8091
		eureka.client.serviceUrl.defaultZone=http://localhost:8090/eureka
		server.port=8079
	
		spring.application.name=employee-zuul-service
	
	updates to consumer (employee-consumer-zuul)
	-	change the producer's name with zuul server's name "employee-zuul-service"
	-	update the API url from /employee to /producer/employee
	
	Restart applications eureka server, producer, zuul server, consumer-zuul	
	