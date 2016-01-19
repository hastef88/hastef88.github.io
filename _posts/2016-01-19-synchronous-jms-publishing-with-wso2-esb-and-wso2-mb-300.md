---
layout: post
title: "Synchronous JMS publishing with WSO2 ESB and WSO2 MB 3.0.0"
description: ""
category: 
tags: []
---
{% include JB/setup %}

The WSO2 ESB provides feature-rich integration with multiple message brokers including our own WSO2 Message Broker, an AMQP/JMS compliant messaging engine. More details of both the products can be found at links [1] and [2].

While article [3] provides information on how to publish asynchronously to a JMS endpoint, there may be instances where you need to make a blocking publish call to the message broker and take alternate actions based on its result. This article guides you on how to do that. Special thanks to Isuru Udana for pointing to this workaround.

### Pre-requisites : 

* Copy the following jar files from the <MB_HOME>/clent-lib folder to the <ESB_HOME>/repository/components/lib folder. 

	* andes-client-3.0.1.jar  
	* geronimo-jms_1.1_spec-1.1.0.wso2v1.jar  
	* log4j-1.2.13.jar  
	* org.wso2.carbon.logging-4.4.1.jar  
	* org.wso2.securevault-1.0.0-wso2v2.jar  
	* slf4j-1.5.10.wso2v1.jar

* Add the connection factories at <ESB_HOME>/repository/conf/jndi.properties file : 

```
connectionfactory.QueueConnectionFactory = amqp://admin:admin@clientID/test?brokerlist='tcp://localhost:5672'
connectionfactory.TopicConnectionFactory = amqp://admin:admin@clientID/test?brokerlist='tcp://localhost:5672'
```

And the jndi mappings for topics used within the proxy. 

```
topic.MyTopicA = MyTopicA
topic.MytopicB = MyTopicB
```


* This sample uses the callout mediator to invoke the synchronized call. Therefore, to enable JMS transport for the callout mediator, enable the following propery in <ESB_HOME>/repository/conf/axis2/axis2_blocking_client.xml file.

```xml
<transportSender name="jms" class="org.apache.axis2.transport.jms.JMSSender"/>
```

* Create a default endpoint with name "defaultEndpoint" from the ESB management console as per article [4].

* Create a custom proxy and use the proxy configuration in below "Sample Proxy" section.

### Sample Request : 

Given the following request payload, the sample proxy will iterate through and publish the full payload to each "xsd:symbol" topic under the "ser:request" section. (Publish to 2 topics MyTopicA and MyTopicB as per the below payload.)

{% highlight xml %}  
<soapenv:Envelope xmlns:soapenv="http://www.w3.org/2003/05/soap-envelope" xmlns:ser="http://services.samples" xmlns:xsd="http://services.samples/xsd">
    <soapenv:Header/>
    <soapenv:Body>
       <ser:getQuote>
          <!--Optional:-->
          <ser:request>
             <!--Optional:-->
             <xsd:symbol>MyTopicA</xsd:symbol>
          	 <xsd:symbol>MyTopicB</xsd:symbol>
            </ser:request>
       </ser:getQuote>
    </soapenv:Body>
 </soapenv:Envelope>
{% endhighlight %}

### Sample Proxy : 

```xml
<?xml version="1.0" encoding="UTF-8"?>
<proxy xmlns="http://ws.apache.org/ns/synapse"
       name="SyncProxy"
       transports="https,http"
       statistics="disable"
       trace="disable"
       startOnLoad="true">
   <target>
      <inSequence>
         <iterate xmlns:xsd="http://services.samples/xsd"
                  xmlns:soapenv="http://www.w3.org/2003/05/soap-envelope"
                  xmlns:ser="http://services.samples"
                  continueParent="true"
                  id="msgIterator"
                  expression="//xsd:symbol"
                  sequential="true">
            <target>
               <sequence>
                  <property name="OUT_ONLY" value="true"/>
                  <property name="topicName"
                            expression="//xsd:symbol"
                            scope="default"
                            type="STRING"/>
                  <log level="custom">
                     <property name="topicNameOfStore" expression="get-property('topicName')"/>
                  </log>
                  <header name="To"
                          scope="default"
                          expression="fn:concat('jms:/', get-property('topicName'),'?transport.jms.ConnectionFactoryJNDIName=TopicConnectionFactory&amp;java.naming.factory.initial=org.wso2.andes.jndi.PropertiesFileInitialContextFactory&amp;java.naming.provider.url=repository/conf/jndi.properties&amp;transport.jms.DestinationType=topic&amp;transport.jms.CacheLevel=producer')"/>
                  <callout endpointKey="defaultEndpoint">
                     <endpoint name="defaultEndpoint">
                        <default/>
                     </endpoint>
                     <source type="envelope"/>
                  </callout>
               </sequence>
            </target>
         </iterate>
         <loopback/>
      </inSequence>
      <outSequence>
         <log>
            <property name="MSG" value="### Messages successfully published. ######"/>
         </log>
         <payloadFactory media-type="xml">
            <format>
               <response xmlns="http://www.test.com/ns">Messages successfully published.</response>
            </format>
            <args/>
         </payloadFactory>
         <send/>
      </outSequence>
      <faultSequence>
         <log>
            <property name="MSG" value="### Broker down ######"/>
         </log>
         <payloadFactory media-type="xml">
            <format>
               <response xmlns="http://www.test.com/ns">Messages did not get published due to a broker validation failure or unavailability.</response>
            </format>
            <args/>
         </payloadFactory>
         <property name="HTTP_SC" value="500" scope="axis2" type="STRING"/>
         <respond/>
      </faultSequence>
   </target>
   <description/>
</proxy>
```

### Things to note : 

1. When using the callout mediator inside iterator mediator, we need to set "continueParent=true" in order to ensure that the execution flow returns to inSequence after the iterator completion.

2. The property "OUT_ONLY" is set per each iteration since the JMS publish operation does not expect a response from the backend. However, any errors/exceptions that could occur during the operation will cause the iterator to stop execution, and fall in to the fault sequence. 

3. The default Endpoint is used within the callout mediator since the destination changes based on the topic name. 

4. The <loopback> mediator is used at the end of the inSequence to direct the flow to the outSequence following a successful invocation. 



[1] : [http://wso2.com/products/message-broker/](http://wso2.com/products/message-broker/)

[2] : [http://wso2.com/products/enterprise-service-bus/](http://wso2.com/products/enterprise-service-bus/)

[3] : [https://docs.wso2.com/display/MB300/Integrating+WSO2+ESB](https://docs.wso2.com/display/MB300/Integrating+WSO2+ESB)

[4] : [https://docs.wso2.com/display/ESB481/Default+Endpoint](https://docs.wso2.com/display/ESB481/Default+Endpoint)
