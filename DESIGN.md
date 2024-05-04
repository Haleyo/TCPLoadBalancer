### TCP Load Balancer Design Considerationg

Please note that this TCP Load Balancer is a simple implementation for evaluative/educational purposes and is not intended to be used in production. 

## Functional Requirements

- Users should be able to initialize a new load balancer given a configuration at start time.
- The load balancer should be able to accept incoming TCP connections from clients.
- The load balancer should be able to forward incoming TCP connections to the backend server with the **least number of active connections**.
- The load balancer client should be able to handle concurrent requests.
- MTLS authentication is a requirement

# Out-of-scope

It is useful to define here what is out-of-scope for this project:

- The load balancer backend server pool is **static**. Node pools are initialized when the load balancer is initialized, there is no funcitonality for adding or removing servers to an existing pool.
- Collection of metrics is out of scope of this project. There is no functionality to collect metrics on the load balancer or the backend servers, with the potential exception of basic health information and number of active connections.
- This solution is not loadtested, and will not necessarily be highly scalable.
- Similarly, no SLA is provided and the solution is not designed with high availability or durability in mind.
- Least connections forwarding is the load balancing strategy implemented, and no other strategies are implemented.
- Timeout settings are not accepted, and the load balancer will not close idle connections automatically
- Only the most basic rate limiting strategy is considered, no other rate limiting startegies are implemented. Rate limiting configuration (capacity and window size) are hardcoded and not configurable. 

# Stretch Goals

- Health checks: No design decisions are made concerning health checks at this time, but a very simple health check strategy will be implemented if time allows (e.g., ping)

## Technical Stack

- The load balancer is implemented in Golang.
- The net library will be used for listening and forwarding TCP connections.
- Cryto library will be used for MTLS authentication.
- We will be using Docker to containerize and run our servers. 

## Proposed Usage and API

The load balancer will be initialized with a static configuration, hardcoded as a struct in the code.
The running count of connections per backend server will be stored in a map, with the server address (or name) as the key and the count as the value, for simplicity. Retrieving the server with the least number of connections could be implemented more efficiently with a heap or priority queue, but for simplicity we will use a map and iterate to find the minimum.
Running the load balancer will require a port mapping as a command line argument to docker, e.g., docker run -p 8080:8080 loadbalancer.

Backend servers will also be Dockerized and will be very minimal in functionality, simply listening on a port and responding with a message, including the name or number ID of the server so that users can easily identify which server they are connected to, (e.g., a 200 status code with the message "Hello world from server 4" for a successful connection). The backend servers similarly will be started with a command line argument specifying the port they are listening on, e.g., docker run -p 8081:8081 backendserver. It may be necessary for the user running the application to start (docker run) all of the backend servers before starting the load balancer.

The library code will have a few simple functions. The parameters supplied in parenthesis are not exaustive.
auth(certificate): This function will authenticate the client using MTLS.

main or initialize: This function will initialize the load balancer with the configuration hardcoded in a struct.

foward(from, to): This function will accept a connection from a client and forward it to the backend server specified in the to argument. Forward will also increment the count of active connections for the backend server, and will check whether the rate limiting capacity has been exceeded.

getServer(): This function will iterate over the map of active connections and return the server with the least number of active connections. Recall that we choose a map for simplicity of implementation but acknowledge that more efficient data structures could be used.

connect(upstream): Connect will copy data from the upstream backend server specified to the client.


## Security Considerations

Security will likely be the last thing to be implemented. Initial iterations will allow any unauthenticated requests. Subsequent pull requests will flesh out the security features.

# Rate Limiting

We will choose a fixed window strategy for rate limiting, as it is the simplest to implement. We will implement a hard rate limiter (as opposed to a soft rate limiter) for simplicity. We will hardcode our capacity variable, as well as hardcode the window size. Connections will be denied with an appropriate status code if the exceed the limit for the window.

# Authentication

To generate and use certificates for MTLS authentication, we will use the crypto library in Golang. For simplicity, we may choose to hardcode the certificate. 

# Authorization

A simple hardcoded map can be used to identify access.

