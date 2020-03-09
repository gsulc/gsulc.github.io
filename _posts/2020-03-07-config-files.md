---
layout: post
title: "Configuration Files"
date: 2020-03-08 23:36:25 -0500
categories: programming design
---
#### A modern examination
Configuration file formatting is an important design consideration. Your choices can make your software easy or difficult to work with. Poor choices can lead to unnecessary support, costing your business. You're likely to be the one who gets stuck answering support questions--while you have other projects to work on. You may have to create extra documentation to explain how to configure your software. Users may avoid your software or jump to another solution as soon as something easier becomes available.

Configuration files should be simple to read, understand and modify. They should be free of clutter and noise. If your users are struggling, you've probably screwed up.

Currently, the choices worth considering are: INI, TOML, and YAML. CSV can also be good in particular situations.

## Good Choices
These formats were specifically *designed* for people to read and modify files easily.
### INI
The **INI** (initialization) format is an *informal* standard that was used heavily by Microsoft from DOS until Windows NT. Linux developers--recognizing the simplicity--continue to use it as a basis for many configuration files. There are many variants with custom features developers needed for their unique application. An INI file is a flat map of key-value pairs that can be grouped into *sections*.

```ini
; comment
pi=3.14

[section A]
mykey=some value

[connection]
ip="127.0.0.1"
port=402
```

If this is all you'll ever need, then INI can be a good choice. If you think you might need more, look to YAML or TOML. Using a *good* standard is better than creating something custom. The work is done for you and other people can recognize and adapt to it. It is extremely unlikely that neither TOML nor YAML don't have the features you want.

### TOML
**TOML** (Tom's Obvious, Minimal Language) is a formal configuration file format. It resembles **INI** with sections and key-value pair as the primary unit and adds support for arrays and dictionaries/maps (called *tables*). Dates are also first-class citizens. Because TOML is built around a hash table, it can't (de)serialize any arbitrary data structure.

```toml
# This is a TOML document.

title = "TOML Example"

[owner]
name = "Tom Preston-Werner"
dob = 1979-05-27T07:32:00-08:00 # First class dates

[database]
server = "192.168.1.1"
ports = [ 8001, 8001, 8002 ]
connection_max = 5000
enabled = true

[servers]

  # Indentation (tabs and/or spaces) is allowed but not required
  [servers.alpha]
  ip = "10.0.0.1"
  dc = "eqdc10"

  [servers.beta]
  ip = "10.0.0.2"
  dc = "eqdc10"

[clients]
data = [ ["gamma", "delta"], [1, 2] ]

# Line breaks are OK when inside arrays
hosts = [
  "alpha",
  "omega"
]
```

### YAML
**YAML** (YAML Ain't Markup Language) is formally defined and supports scalars, collections, and maps/dictionaries. It also supports references and non-common objects. With more features than anything else, implementations are more complex.

YAML is a super-set of JSON. *Tags* allow you to (de)serialize non-common objects (objects other than numbers, strings, booleans, and time). *Anchors* (i.e. labels) let you re-use objects by reference within a document.

```yaml
# created by http://yaml.org/xml/xml2yaml.xsl
--- %YAML:1.0
number: 34843
date: '2001-01-03'
bill-to: &id001 
   given: 'Chris'
   family: Dumars
   address: 
      lines: |
         458 Walkman Dr.
          Suite #292
          
      city: 'Royal Oak'
      state: MI
      postal: 48046
ship-to: *id001 
product: 
   - 
      sku: BL394D
      quantity: 4
      description: Basketball
      price: '450.00'
   - 
      sku: BL4438
      quantity: 1
      description: 'Super Hoop'
      price: '2392.00'
tax: '251.42'
total: '4443.52'
comments: 'Late afternoon is best. Backup contact is Nancy Billsmer @ 338-4338'
...
```

#### Warning
These features means utilizing YAML improperly in your code can cause security holes for code execution. Additionally, there are slight inconsistencies across implementations due to complexity and ambiguity in the standard. Some languages might have restrictions that prevent some YAML from working. For example, Python cannot use lists as a dictionary key. Some of the more complex features can also make it difficult to read. You can [read more](https://www.arp242.net/yaml-config.html) some of YAML's shortcomings.

### CSV
**CSV** (or a delimited file) is a good choice for describing many instances of the same object. Take caution: it can be painful if you need to change the structure. They can be a good alternative to a database: offering easier access to make changes.

Imagine 50+ entries in YAML:
```yaml
- name: John Doe
  phone: "123-456-7890"
  email: john@doe.com
- name: Jane Doe
  phone: "098-765-4321"
  email: jane@doe.com
```
compared to CSV:
```csv
name,phone,email
John Doe,"123-456-7890",john@doe.com
Jane Doe,"098-765-4321",jane@doe.com
```
Less repeated information means less noise. But it also means less flexibility. What if you want a new property that only applies to some entries? Excel and  plug-ins for other programs let you view CSV files as a table. If you make a *Gist* on Github, it will do this.

## Poor Choices
### JSON
JSON is less readable than TOML and YAML. Excessive quotations, curly braces, and commas get in the way of clarity. These are noise that users have to filter out. It is *unprofessional* to force them to do so--especially when libraries are so trivial to use. JSON is simple to implement, making it excellent for data serialization--particularly in web applications where JSON is native to JavaScript. Note that it is limited to basic, common types: strings, integers, floating points, booleans. JSON is not the worst choice for configuration.

### XML
XML is a poor choice for configuration files. Noise outweighs the information, even when using *attributes*. Human users get frustrated with it. Utilizing attributes means altering or designing your source code to tailor the configuration behavior. Yuck! XML serializers don't support dictionaries, so you have to write code if you want to use them. There are mechanisms for serializing non-common objects (without dictionaries). The excess noise also makes data transfer/storage bloated, unless you use attributes. XML doesn't seem to be much good at anything.

#### But XML is highly standardized!
Large organizations make mistakes and some are bound by regulations. SOAP was so unwieldy, it is was on its death throes not long after better alternatives came about. Dealing with XML configuration files is unpleasant. [WCF configuration was torturous--even for people who really liked the technology](https://www.youtube.com/watch?v=76X9oo-LlUY&t=8m22s)! The .NET Core switch to JSON application configuration was a significant improvement, but there were two widespread, better choices available. (At least there is support for INI.) YAML was released over a year before .NET was, so the engineers could have chosen it from the beginning--in theory. Of course time constraints, popularity, and lack of information exchange get in the way.

Let's look at a short example of a WCF service configuration:

```xml
<configuration>
  <system.serviceModel>
    <behaviors>
      <serviceBehaviors>
        <behavior name="MyServiceBehavior">
          <serviceMetadata httpGetEnabled="true" />
        </behavior>
      </serviceBehaviors>
    </behaviors>
    <services>
      <service name="MyNamespace.MyService"
               behaviorConfiguration="MyServiceBehavior">
        <endpoint address="/Address1"
                  binding="basicHttpBinding" 
                  contract="MyNamespace.IMyService"/>

        <endpoint address="mex"
                  binding="mexHttpBinding"
                  contract="IMetadataExchange" />
      </service>
    </services>
  </system.serviceModel>
</configuration>
```
and re-imagine it in YAML:
```yaml
- !System.ServiceModel
  behaviors:
    serviceBehaviors:
      - &myServiceBehavior !ServiceBehavior
        serviceMetadata:
          httpGetEnabled: true
  services:
    - name: MyNamespace.MyService
      behavior: *myServiceBehavior
      endpoints:
        - address: /Address1
          binding: basicHttpBinding
          contract: MyNamespace.IMyService
        - address: mex
          binding: mexHttpBinding
          contract: IMetadataExchange
```

Not only is the YAML significantly shorter and easier to read, but we have the ability do deserialize `UE.Samples.HelloService` service's `behavior` to reference the created object directly. You could also have it point to the same string data and not use the tags to create objects. In the XML implementation, the developers had to create extra code to wire things together by name.

## Closing
Make sure your choice is right. Learn from the mistakes of major platforms; don't follow their lead. Look at a wide array of projects to see what they do; understand the choices and what makes them successful. It's easy to get stuck in a bubble.

YAML is the most powerful choice with the most features and flexibility, but you may want to only use a subset for simplicity. Also take caution to use it safely. Depending on your goals and constraints, INI, TOML, or CSV may be a better choice for you. JSON is not great, but not nearly as bad as XML. Read more and try them out before deciding.
