# Introduction #

Guice's emphasis on binding constants using binding annotations rather than Strings is great, but it can be very tempting to slip back into the habit of using Named annotations. GuiceBox makes it slightly easier to be good, by allowing constants to be automatically bound to annotations based on `.properties` files in your environment.

More importantly, the importing of properties files is username-dependent. This makes it really easy to set up different profiles for each of your environments.

# How to Use #

## Wire the PropertiesModule ##
Firstly, you'll need to wire a `PropertiesModule` instance to GuiceBox like this:
```
final GuiceBox guicebox = GuiceBox.init(new PropertiesModule());
```
This will set up GuiceBox to automatically detect `.properties` files in your environment.

## Create Binding Annoations ##
Secondly, you'll want to create some binding annotations for your constants. There are two patterns for this: required, and optional.

### Optional Properties ###
A property is optional when you can provide a _reasonable default_ value, reducing the burden on your API users. To do this, use `@Inject(optional = true)`, like this:
```
// The multicast port for sending heartbeats
private int _destPort = 9797;
@Inject(optional = true) final void setDestinationPort(@DestinationPort int port)
{
	_destPort = port;
}
```
This way, your API users can set the value if they want, but if they don't it will use the default value you have provided. (Be sure to provide appropriate concurrency controls if necessary.)

### Required Properties ###
A property is required if it is not useful or practical to provide a default value. To do this, use a `final` field and inject into the constructor, like this:
```
// The multicast address for heartbeats
private final InetAddress _groupAddress;

@Inject UdpTransport(@GroupAddress String groupAddress) throws UnknownHostException
{
	_groupAddress = InetAddress.getByName(groupAddress);
}
```
This way, if the environment doesn't contain a property assignment for the annotation, Guice will not start the application.

## Create `.properties` Files ##

PropertiesModule searches for `.properties` files using the following precedence:
  1. `./properties/`_username_`/*.properties`
  1. `./properties/*.properties`

The idea here is to put property assignments for common values in `./properties/`, and put environment-specific values in `./properties/`_username_`/`. The benefit of this is that you can set up a separate 'user' account for each of your environments - say, "dev", "uat", "prod" - and then structure your `properties` and application directories like this:
```
~/mycoolapp/properties/cool_common.properties
~/mycoolapp/properties/dev/cool_dev.properties
~/mycoolapp/properties/uat/cool_uat.properties
~/mycoolapp/properties/prod/cool_prod.properties
~/mycoolerapp/properties/cooler_basic.properties
~/mycoolerapp/properties/dev/cooler_dev.properties
~/mycoolerapp/properties/uat/cooler_uat.properties
~/mycoolerapp/properties/prod/cooler_prod.properties
...etc
```
This lets you distribute the whole application directory, including all properties files for all environments, and know that as long as the application is run as the appropriate 'user', it will use the right settings. This makes deployment a lot easier.

Another important point to note is that you can have as many properties files as you want. GuiceBox doesn't care how you split your values between multiple `.properties` files - it will just load all of them. If you declare the same property in both `./properties/hello.properties` and `./properties/`_username_`/world.properties`, it will use the one from `world.properties`.

## Contents of Properties Files ##

There are two types of entries (other than comments) in properties files: those that will be bound to binding annotations (preferred), and those that will be bound to `@Named` annotations. They look like this:
```
# Used in sample application TestMain
org.guicebox.Param1=base1
org.guicebox.Param2=base2
org.guicebox.Parm3=base3
non.annotation.property=Not an annotation
```
Note that the fully-qualified name of the binding annotation is required to bind a constant to Guice. Anything that doesn't match the FQN of a binding annotation will just be loaded into the `@Named` namespace. If you make a typo with the FQN, PropertiesModule will tell you when you start the app (see below).

# Properties Log Report #

When you start your application, PropertiesModule will [log](LoggingBridge.md) the values of all your properties, so you can easily check how it has interpreted your properties files. The output looks like this:
```
   @org.guicebox.failover.udp.GroupAddress                      = 239.192.169.151
   @org.guicebox.failover.Environment                       = DEV
    my.secret.password                                = *****
   @org.guicebox.failover.WellKnownAddress                  = 192.168.1.100
    non.annotation.property                           = Not an annotation
    org.guicebox.Parm3                                = base3
   @org.guicebox.Param2                               = username2
   @org.guicebox.Param1                               = username1
```
Notice that properties that have been bound to annotations include an `@` symbol on the left. Here we can see the typo I made in `Param3` has caused this to not be bound to the annotation properly. This line is output to `stderr` so in most IDEs this will also show in red.

Notice also that the value for `my.secret.password` has been masked. Any property that ends in "password" (ignoring case) will be masked in this way. If your organisation has an application security team that is required to supply passwords, you can have them copy a `password.properties` file into your environment for GuiceBox to read. Development staff can then analyse log files without risk of seeing the password.

# Log Header #

Another cute feature of PropertiesModule is its ability to output an ASCII-Art logo or some kind of boilerplate text to the beginning of your logs. Just put a file called `loghead.txt` somewhere in your classpath and it will be output as soon as Guice calls PropertiesModule to configure bindings.