# Keycloak Email Provider for AWS SES (Simple Email Service)

This is a drop-in Email Provider SPI replacement implementation for the default SMTP one provided by [Keycloak SSO](https://www.keycloak.org) server.

This is forked from the original implementation by [dasniko](https://github.com/dasniko/keycloak-aws-ses-email-provider).  
The only changes from the original implementation are to be updated to work with KeyCloak 25 and Java 21.

Niko's original warnings apply:
> It's for demo purposes only and can be used as base for your own implementation.
> 
> The codebase is provided _as-is_ and might not be free of errors.
> So, you're on your own when using it.
> Generally, this project is _work in progress_.

## Dependencies

The implementation uses the AWS Java API Version 2.
This dependency will be packaged with the help of the Maven Shade Plugin into the target JAR archive.

To save space and build a smaller fat-jar, all the async resources have been excluded from the AWS SDK.
The email provider just uses the synchronous client.

All other dependencies are used from Keycloaks underlying Quarkus server
(see [jboss-deployment-structure.xml](./src/main/resources/META-INF/jboss-deployment-structure.xml)).

## Installation

Build the project with `mvn package` and copy the generated `.jar` file (the shaded one, not the original one!)
into the `standalone/deployments/` folder of your Keycloak installation.
It will be deployed automatically (hot deployment works the same).

## Configuration

There are several ways SPIs can be configured now with KeyCloak 25.

### 1. `kc.sh`

To configure the SPI using either the `kc.sh build` or `kc.sh start` commands, it is configured in the same way as other providers.  

```bash
/opt/keycloak/bin/kc.sh build \
  --spi-emailSender-provider=aws-ses \
  --spi-emailSender-aws-ses-enabled=true;
```

or;

```bash
/opt/keycloak/bin/kc.sh start \
  --spi-emailSender-provider=aws-ses \
  --spi-emailSender-aws-ses-enabled=true;
```

### 2. Configuration File

To configure the email provider SPI, include a snippet like this in your `standalone(-ha).xml` file:

```xml
<subsystem xmlns="urn:jboss:domain:keycloak-server:1.1">
  ...
  <spi name="emailSender">
    <default-provider>aws-ses</default-provider>
    <provider name="aws-ses" enabled="true">
      <!-- Optional, if you want (or need) to set another region for SES as the environment default region: -->
      <properties>
        <property name="region" value="eu-west-1"/>
      </properties>
    </provider>
  </spi>
  ...
</subsystem>
```

### 3. JBoss CLI
Alternatively, you can use this `jboss-cli` script snippet to configure your Keycloak server:

```
if (outcome == success) of /subsystem=keycloak-server/spi=emailSender:read-resource
  /subsystem=keycloak-server/spi=emailSender/:remove()
end-if
/subsystem=keycloak-server/spi=emailSender/:add(default-provider=aws-ses)
/subsystem=keycloak-server/spi=emailSender/provider=aws-ses/:add(enabled=true)
# Optional, if you want (or need) to set another region for SES as the environment default region:
# /subsystem=keycloak-server/spi=emailSender/provider=aws-ses/:write-attribute(name=properties,value={"region" => "eu-west-1"})
```

### Configuration Notes
As the Email Provider SPI is not selectable/configurable on a per-realm base, you can't set the AWS SES provider for one realm and leave the default SMTP provider in another.
If you use/configure this SPI to be used in Keyclaok, it's system-wide!

Additionally, Keycloak does not provide a possibility to configure an Email Provider SPI through the admin console with custom values.
The way described above is the only way.

However, with this SPI implementation, you can use the values for `from`, `fromDisplayName`, `replyTo` and `replyToDisplayName` from the defaut SMTP configuration page in your sent emails:

![](img/config.png)

Will result in:

![](img/email.png)


## AWS Configuration

This SPI makes use of the [DefaultAWSCredentialsProviderChain](https://docs.aws.amazon.com/AWSJavaSDK/latest/javadoc/com/amazonaws/auth/DefaultAWSCredentialsProviderChain.html)
and the [DefaultAWSRegionProviderChain](https://docs.aws.amazon.com/AWSJavaSDK/latest/javadoc/com/amazonaws/regions/DefaultAwsRegionProviderChain.html).
So, it's on you how you configure your Keycloak environment in a way that is able to authenticate itself against AWS.

For specifying a different region for SES usage, see configuration above.

Your used profile needs the privilege to send emails with at least `ses:SendEmail`.
