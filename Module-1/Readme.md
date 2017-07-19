# Create a Simple LoadBalancing Stylebook

If you followed along in the [Introduction](../) you would have noticed that the compiled stylebook (shown [here](../code/lb-only-stylebook.yaml)) is more or less the equivalent to the NS CLI command : 

```
add lb vserver ${vServer-Name} HTTP ${VIP} ${PORT} -lbmethod ${ROUNDROBIN || LEASTCONNECTION}
```

What we need is to do next is dynamically add back-end hosted services that the Load Balancer can front end. We need a stylebook which incorporates the addition of the following NS CLI commands to our logic: 

```
add server ${Server-Name} ${IP-Address}
add service ${Service-Name} ${Server-Name} HTTP ${PORT} 
bind lb vserver ${vServer-Name} ${Service-Name}
```

### Step 1: Open the `lb-only-stylebook.yaml` in the IDE editor
___

In your Clou9 IDE, navigate and open the file by double clicking [`/workspace/MAS-Stylebook-101/code/lb-only-stylebook.yaml](../code/lb-only-stylebook.yaml) in the editor pane. 

Once open, save the file as `lb-srvc-stylebook.yaml` in the same directory. We will edit this version to add additional functionality to the styebook. 

### Step 2: Add the following to `Parameters` for additional input
___

The **Parameters** YAML block looks like the following: 

```yaml
parameters:
  -
    name: name
    type: string
    label: Application Name
    description: Name of the application configuration.
    required: true
  -
    name: ip
    type: ipaddress
    label: Application Virtual IP (VIP)
    description: Application VIP that the clients access.
    required: true
  -
    name: lb-alg
    type: string
    label: LoadBalancing Algorithm 
    description: Choose the load balancing method for LB client requests.
    allowed-values:   
        - ROUNDROBIN
        - LEASTCONNECTION
    default: ROUNDROBIN
```

Update the YAML text by copying and pasting the content following `## Additional Content Update` so the stylebook can collect additional inputs from end user to configure back-end services. 

```yaml 
parameters:
  -
    name: name
    type: string
    label: Application Name
    description: Name of the application configuration.
    required: true
  -
    name: ip
    type: ipaddress
    label: Application Virtual IP (VIP)
    description: Application VIP that the clients access.
    required: true
  -
    name: lb-alg
    type: string
    label: LoadBalancing Algorithm 
    description: Choose the load balancing method for LB client requests.
    allowed-values:   
        - ROUNDROBIN
        - LEASTCONNECTION
    default: ROUNDROBIN

## Additional Content Update
  -
    name: svc-servers                            # Name of the first additional parameter input
    type: object[]                               # This input will be a dynamic list of objects, in our case a list of 'servers' with the following parameters: "ip" and "svc-name"  
    label: Application Server IPs                # Display text on MAS GUI for parameter input
    description: Backend host Server IPs         # Parameter details 
    required: true                               # Marked as a required input from end user
    parameters:                                  # Nested parameters that define objects in the list of objects for "svc-server" input parameter
      - 
        name: ip                                 # Name of first parameter for an object in "svc-servers" 
        label: IP Address of the Server          # Display text on MAS GUI for parameter input
        description: IP Address of the Server    # Parameter details
        type: ipaddress                          # Type of formatted input expected: IP address "192.168.10.10" for example.
        required: true                           # Marked as a required input from end user
      - 
        name: svc-name                           # Name of second parameter for an object in "svc-servers" 
        label: Name of the service               # Display text on MAS GUI for parameter input
        description: Name of the service         # Parameter details 
        type: string                             # Type of formatted input expected: "String" for example
        required: true                           # Marked as a required input from end user
  -
    name: svc-port                               # Name of the second additional parameter input
    type: tcp-port                               # Type of formatted input expected: Port "8080" for example
    label: Service Port                          # Display text on MAS GUI for parameter 
    description: TCP port of back-end service    # Parameter details 
    default: 80                                  # Default port value if one is not specified. 
```
Once completed, the stylebook in its current state now will collect the following information: 

GUI Input Label | Translated NetScaler Configuration
--- | ---
Application Name | LB vServer Name
Application Virtual IP (VIP) | LB vServer VIP
LoadBalancing Algorithm | LoadBalancing Method
Application Server IPs | List of Server Names and IPs
Service Port  | Server Port

Lastly, we have to update the [Component](../#Components) YAML block to add logic that takes in these parameters to add servers, services, and bind services to lb vServers for a functional Load balancer. 

### Step 3: Add the following to `Component` to configure Servers and Services
___

The **Component** YAML block looks like the following: 

```yaml
components:
  -
    name: lbvserver-comp
    type: ns::lbvserver
    description: A Builtin Nitro StyleBook to build lbvserver object.
    properties:
      name: $parameters.name
      servicetype: HTTP
      ipv46: $parameters.ip
      port: 80
      lbmethod: $parameters.lb-alg
```
Update the YAML text by copying and pasting the content following `## Additional Content Update` so the stylebook can configure additional configurations on the NetScaler ADC to define LoadBalancing services and back end servers. 

>**Note:** Components can be nested within each other to dictate order of operations. For example, only a single NITRO command can create a lb vserver or an lb service, and only 1 other NITRO command can bind the two. To create a 1 active vserver, you would need a minimum of 3 API calls. Due to the NITRO API, you can define components within components to execute the parent component first then subsequent child components. See example below where `type: ns::lbvserver_service_binding` is a child of `type: ns::service`, and `type: ns::service` is a child of `type: ns::lbserver`. Therfore the order of component creation is as follows: `type: ns::lbserver` then `type: ns::service` and lastly `ns::lbvserver_service_bindin`.

```yaml
components:
  -
    name: lbvserver-comp
    type: ns::lbvserver
    description: A Builtin Nitro StyleBook to build lbvserver object.
    properties:
      name: $parameters.name
      servicetype: HTTP
      ipv46: $parameters.ip
      port: 80
      lbmethod: $parameters.lb-alg

## Additional Content Update
    components:                                      # Notice, this is a component (ns::service) under a component under (ns::lbvserver) which means it executes after the prior succeeds. a service is created with a lb vserver is made. 
      -
        name: svc-comp                               # Name of component for reference
        type: ns::service                            # This defines the type of component from the imported stylebook referred to by the alias `ns`. Corresponds to built-in NITRO API
        repeat: $parameters.svc-servers              # This defines that the following parameter "svc-servers" which was defined a list of object will be repeated with the following component and properties. Think of this as a "for" loop in programming.
        repeat-item: srv                             # The alias used to refer to one object in "svc-servers"
        properties:                                  # This defines the needed values of "type: ns::service"
          name: svc- + str($srv.svc-name)            # Service Name with naming convention "svc-{name parameter in server object}""
          servicetype: HTTP                          # Hardcoded service type
          ip: $srv.ip                                # Required input when defining a NS service derived by the IP parameter of the server object.
          port: $parameters.svc-port                 # Required input when defining a NS service derived from the "svc-port" parameter.
        components:                                  # Notice, this is a component (ns::lbvserver_service_binding) under a component under (ns::service) which means it executes after the prior succeeds. A service is bound to a lb vserver once the service is made.
          -
            name: member-bind                        # Name of component for reference
            type: ns::lbvserver_service_binding      # This defines the type of component from the imported stylebook referred to by the alias `ns`. Corresponds to built-in NITRO API
            properties:                              # This defines the needed values of "type: ns::lbvserver_service_binding"
              name: $parent.parent.properties.name   # Name of the vserver to bind to. Value here is derived by the grand-parent component's value for "name" 
              servicename: $parent.properties.name   # Name of the service to bind to. Value here is derived by the parent component's value for "name" 
```

Once completed, you can copy and paste all the content in `lb-only-stylebook.yaml` into [YAML Lint](http://www.yamllint.com/) syntax checker to validate your script. In its current state, the stylebook will now do the following things in the following order: 

1. Create a LB vServer with:
  * A lb vserver **name**
  * A **VIP** endpoint
  * A default protocol of **HTTP** 
  * Load Balancing method of **ROUNDROBIN** or **LEASECONNECTION**

2. Create a Service + Servers with:
  * Bound **back-end servers**
  * Defined service **port**
  * Derived **service name** 
  * Service default **HTTP** Protocol
  
3. Bind the Service to the LB vServer

### Step 4: Deploy the Stylebook via MAS GUI

