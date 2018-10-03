# Spring Boot Cloud example

## Netflix Eureka : Service discovery and registry

 Application's related properties are defined in individual properties file or hardcoded into code. If a change is required then it becomes very cumbersome to adapt the changes if there are multiple applications
 
 With Netflix Eureka server, hardcoding can be removed (e.g port details in producer-consumer example)
 If a service is changed like its port or location, dependent services are not impacted if services are using Eureka service discovery and registry
 
Implementation of Eureka  Service discovery and registry
 1. Develop two services producer and consumer (Rest Application)
 2. Implement service registration for Producer application
 3. Implement service discovery for Consumer application
 
Eureka server implmentation
 Dependency management configuration in pom for spring-cloud-dependencies. The reason is to effectively adapt to the latest changes/releases made by Netflix team
 
Updates to Producer (Service Registry)
 1. add dependency spring-cloud-starter-eureka and dependency management spring-cloud-dependencies in the pom
 2. add @EnableDiscoveryClient in the SpringBootApplication
 3. add application.property to refer the eureka server running on 8090 "eureka.client.serviceUrl.defaultZone=http://localhost:8090/eureka"
 4. Restart the producer application. An "UNKNOWN" application will be registered now at Eureka server "http://localhost:8090/"
 5. Create a bootstrap.properties to provice a unique name to this application and restart it "spring.application.name=employee-producer"
 6. Restart the producer application. An "employee-producer" application will be registered now at Eureka server "http://localhost:8090/"

Updates to Consumer (Service Discovery) 
 1. add dependency spring-cloud-starter-eureka and dependency management spring-cloud-dependencies in the pom
 2. add @EnableDiscoveryClient in the SpringBootApplication for its own discovery however its not required for dicovering producer
 3. add application.property to refer the eureka server running on 8090 "eureka.client.serviceUrl.defaultZone=http://localhost:8090/eureka"
 4. Create a bootstrap.properties to provice a unique name to this application and restart it "spring.application.name=employee-consumer"
 5. Add below code to find the URI of producer using DiscoveryClient and use the URL to invoke producer
	 
	 	@Autowired
		DiscoveryClient discoveryClient;
		public void getEmployee() throws RestClientException, IOException {
		
			List<ServiceInstance> instances=discoveryClient.getInstances("employee-producer");
			ServiceInstance serviceInstance=instances.get(0);			
			String baseUrl=serviceInstance.getUri().toString() + "/employee";
			
 6. Restart the producer application. An "employee-consumer" application will be registered now at Eureka server "http://localhost:8090/" and this should be able to discover producer via the unique id
 
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