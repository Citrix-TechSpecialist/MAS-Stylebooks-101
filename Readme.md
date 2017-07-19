# Table of Contents

1. [Module 0: Access your Development IDE](./Module-0)
2. [Module 1: Create a Simple LoadBalancing Stylebook](./Module-1)
3. [Module 2: Creating Composite Content Switching Stylebook](./Module-2)

# Introduction 

[StyleBooks](http://docs.citrix.com/en-us/netscaler-mas/12/stylebooks.html) simplify the task of managing complex NetScaler configurations for your applications. A StyleBook is a [YAML](https://www.mirantis.com/blog/introduction-to-yaml-creating-a-kubernetes-deployment/) defined template that you can use to create and manage NetScaler configurations. You can create a StyleBook for configuring a specific feature of NetScaler ADC via NetScaler MAS, or you can design a StyleBook to deploy complex configurations for an enterprise application deployment such as Microsoft Exchange or Lync.

# Pre-requisites 

Stylebooks are a feature of [NetScaler MAS](http://docs.citrix.com/en-us/netscaler-mas/12.html) that can configure [NetScaler ADC's](http://docs.citrix.com/en-us/netscaler/12.html) to a desired state based on input YAML files. [YAML](https://learn.getgrav.org/advanced/yaml) is very sensitive to case and spaces therefore it is highly recommended that you use a formatted syntax highlighting rich text editor like [Sublime Text 3](https://www.sublimetext.com/). In this tutorial however, we will be leveraging [Cloud9's IDE](https://c9.io/) to view and edit YAML files along with execute python code to issue NITRO API calls to MAS. In summary, at a bare minimum, you should have: 
   
   * NetScaler MAS
   * NetScaler ADC (VPC, CPX, SDX, or MPX)
   * Sublime Text 3
   * General knowledge of [NetScaler's NITRO API](http://docs.citrix.com/ja-jp/netscaler/11/nitro-api.html)

# Introduction: Anatomy of a [Stylebook](https://docs.citrix.com/en-us/netscaler-mas/11-1/stylebooks.html) 

Here is an [example of a Stylebook](./code/lb-only-stylebook.yaml) written in YAML for reference when discussing the basics of constructing a Stylebook.

Stylebooks are broken down to [several logical sections](http://docs.citrix.com/en-us/netscaler-mas/11-1/stylebooks/stylebooks-grammar.html), some of the main ones are listed below: 

  1. [Header](http://docs.citrix.com/en-us/netscaler-mas/11-1/stylebooks/stylebooks-grammar/header-section.html)
  2. [Imports](http://docs.citrix.com/en-us/netscaler-mas/11-1/stylebooks/stylebooks-grammar/import-stylebooks-section.html)
  3. [Parameters](http://docs.citrix.com/en-us/netscaler-mas/11-1/stylebooks/stylebooks-grammar/parameters-section.html)
  4. [Components](http://docs.citrix.com/en-us/netscaler-mas/11-1/stylebooks/stylebooks-grammar/components.html) 
  5. [Outputs](http://docs.citrix.com/en-us/netscaler-mas/11-1/stylebooks/stylebooks-grammar/outputs.html)  

In the following sections, we will see samples for each section defined above. The objective of the stylebook discussed below is to create a simple Load Balancing (LB) vServer, for learning purposes. The code below does not bind any back-end services to the LB vServer. This will be done in [Module 1](./Module-1).

### [Header](http://docs.citrix.com/en-us/netscaler-mas/11-1/stylebooks/stylebooks-grammar/header-section.html)
This section lets you define the identity of a StyleBook and describe what it does. 
  > This is a mandatory section

  ```yaml
  name: basic-lb-config                           # A name for this StyleBook.                                    
  description: "A simple LB configuration."       # A description defining what this StyleBook does. This description appears on the NetScaler MAS GUI.
  display-name: "LB StyleBook (HTTP)"             # A descriptive name for the StyleBook that appears on the NetScaler MAS GUI.
  author: Mike Smith                              # Metadata of who created this Stylebook
  namespace: com.example.stylebooks               # A namespace forms part of a unique identifier for a StyleBook to avoid name collisions.
  schema-version: "1.0"                           # Stylebook's grammar version. As of right now, it always takes the value “1.0” for MAS 12.0 release.
  version: "0.1"                                  # The version number of the StyleBook. You can change the version number when you update the StyleBook.
  ```	

### [Imports](http://docs.citrix.com/en-us/netscaler-mas/11-1/stylebooks/stylebooks-grammar/import-stylebooks-section.html)

This section lets you declare which other StyleBook you want to refer to from your current StyleBook. Importing NetScaler NITRO configuration StyleBooks or other StyleBooks is required to write a StyleBook. The `netscaler.nitro.config` Stylebook is imported into MAS by default to reference NITRO, all subsequent Stylebooks being referred to must be first imported into MAS before being references by newer Stylebooks. 
  > This is a mandatory section.

**Example:** This defines the import of `netscaler.nitro.config` base stylebook as a dependency for the creating of the Stylebook. 

  ```yaml
  import-stylebooks:                     # This defines the start of the Import Stylebooks YAML block
   -
    namespace: netscaler.nitro.config    # Every StyleBook must refer to the netscaler.nitro.config namespace if it uses any of the NITRO configuration objects directly, otherwise refer to the namespace in the header of the Stylebook you are looking to import.
    prefix: ns                           # This can be used as an alias when refering to the given namespace in the follow sections. 
    version: "10.5"                      # This specifies the version of the stylebook referred to in the imported Stylbook's header.
  ```

### [Parameters](http://docs.citrix.com/en-us/netscaler-mas/11-1/stylebooks/stylebooks-grammar/parameters-section.html)

This section lets you define all the parameters that you require in your StyleBook to create a configuration. It describes the input that your StyleBook takes. Although this is an optional section, but most StyleBooks might need one. You can consider the parameters section to define the questions you want users to answer when they use the StyleBook to create a configuration on a NetScaler instance.

When you import your StyleBook into NetScaler MAS and use it to create a configuration, the GUI uses this section of the StyleBook to display a form that takes input for values of the parameters you have defined.

**Example:** This shows three defined input parameters required from the end user.

  ```yaml
  parameters:                                                 # This defines the start of the Input Values required for this Stylebook.
  -
    name: name                                                # Name of the first input parameter.
    type: string                                              # Type of formatted input required.
    label: Application Name                                   # Text that represented this input in MAS' GUI.
    description: Name of the LB vServer.                      # Details of the input for end user context.
    required: true                                            # Specifies if a mandatory parameter 
  -
    name: ip                                                  # Name of the second input parameter.
    type: ipaddress
    label: Application Virtual IP (VIP)
    description: Application VIP that the clients accesses.
    required: true
  -
    name: lb-alg                                               # Name of the third input parameter.
    type: string
    label: LoadBalancing Algorithm 
    description: Choose the method for LB client requests.
    allowed-values:                                            # This key allows you to provide a list of options for selection as an input to this parameter. 
        - ROUNDROBIN                                           # List of individual selection options. 
        - LEASTCONNECTION
    default: ROUNDROBIN                                        # If `allowed-values` is present, then `default` will specify the default selection value from the `allowed-values`list. 

  ```

### [Components](http://docs.citrix.com/en-us/netscaler-mas/11-1/stylebooks/stylebooks-grammar/components.html) 

The Components construct in a StyleBook is considered as the most important section in the StyleBook. In this section, you define the configuration objects that have to be created. Using this construct, you can build one or multiple configuration objects of the same type.

The components construct can use the input provided in the parameters section to adapt the configuration generated by the StyleBook. This is an optional section, although most StyleBooks have a components section.

>Note: Components can be nested within each other as parent - child nodes in YAML. Nesting of individual components dictates order of operations. For more details and an example, refer to [Module-1](./Modules-1) footnote in **Step 3**. 

**Example:** This example contains only *one* component for simplicity.

  ```yaml
  components:                                                   # This defines the start of the components used to build the logic required for this Stylebook.
  -                                                             # Delimiter for every new component.
    name: lbvserver-comp                                        # This defines the name of the component.
    type: ns::lbvserver                                         # This defines the type of component from the inported stylebook refered to by the alias `ns`
    description: From Nitro StyleBook, makes lbvserver object.  # Details of the component for end user context.
    properties:                                                 # The properties defined here are the attributes of the "lbvserver" resource from NetScaler's NITRO REST API.
      name: $parameters.name                                    # This provides an input value for the property of lbvserver from the parameter first parameter `name` defined above. 
      servicetype: SSL                                          # In this example, this input value `servicetype` for this component is hardcoded with `SSL`
      ipv46: $parameters.ip                                     # This provides an input value for the property of lbvserver from the parameter second parameter `ip` defined above. 
      port: 80                                                  # In this example, this input value for `port` is hardcoded with `80`         
      lbmethod: $parameters.lb-alg                              # This provides an input value for the property of lbvserver from the parameter second parameter `lb-alg` defined above. 

  ```

### [Outputs](http://docs.citrix.com/en-us/netscaler-mas/11-1/stylebooks/stylebooks-grammar/outputs.html)

In the outputs section, you specify what a StyleBook exposes to its users after it has completed creating all the configuration objects successfully. The outputs section of a StyleBook is optional. A StyleBook does not need to return outputs. However, by returning some internal components as outputs, it allows any StyleBooks that import it more flexibility as you can see when creating a composite StyleBook.

**Example:** Here we provide the only output being the lbvserver object created by the component section. This can then later be used by other stylebooks as inputs for other components. 

  ```yaml 
  outputs:                                             # This defines the start of the Output values generated by this Stylebook.                             
  -                                                    # Delimiter for every new output.
    name: lbvserver-comp                               # Name of the output parameter
    value: $components.lbvserver-comp                  # Object as the value for the output. In this case all properties and values for the `lbveserver-comp` from the `components` section above. 
    description: Builds the Nitro lbvserver object     # Details of the input for end user context.
  ```