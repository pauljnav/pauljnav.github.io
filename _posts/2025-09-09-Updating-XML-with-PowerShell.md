# Updating XML with PowerShell

In this example, we need to adjust logging levels without manually editing XML. Doing this with PowerShell is powerful, especially when you want **dynamic, maintainable updates**.
You can also change the other element settings, and I provide a more complex example having many additional nodes and elements at the end of the blog where you can try out more processing.

A typical log4j XML file may look like this:

```xml
<?xml version="1.0" encoding="UTF-8"?>

<log4j:configuration xmlns:log4j="http://logging.apache.org/">

  <appender name="App1" class="org.apache.log4j.rolling.RollingFileAppender">
    <rollingPolicy class="org.apache.log4j.rolling.FixedWindowRollingPolicy">
      <param name="fileNamePattern" value="D:\Logs\App1%i.log"/>
      <param name="maxIndex" value="5"/>
    </rollingPolicy>
    <triggeringPolicy class="org.apache.log4j.rolling.SizeBasedTriggeringPolicy">
      <param name="maxFileSize" value="50MB"/>
    </triggeringPolicy>
  </appender>

  <appender name="App2" class="org.apache.log4j.rolling.RollingFileAppender">
    <rollingPolicy class="org.apache.log4j.rolling.FixedWindowRollingPolicy">
      <param name="fileNamePattern" value="D:\Logs\App2%i.log"/>
      <param name="maxIndex" value="3"/>
    </rollingPolicy>
    <triggeringPolicy class="org.apache.log4j.rolling.SizeBasedTriggeringPolicy">
      <param name="maxFileSize" value="20MB"/>
    </triggeringPolicy>
  </appender>

  <logger name="Logger1">
    <level value="info"/>
    <appender-ref ref="App1"/>
  </logger>

  <logger name="Logger2">
    <level value="info"/>
    <appender-ref ref="App2"/>
  </logger>

</log4j:configuration>
```
### Objective
To programmatically update the logger `Logger1` from `info` to `debug`.

#### Load the xml document - Preserving Whitespace

```powershell
# Load XML
$XmlPath = "\path\to\xmlfile.xml"
$xml = New-Object System.Xml.XmlDocument
$xml.PreserveWhitespace = $true
$xml.Load($XmlPath)
```

#### Preserving Whitespace
Many XML files include human-readable comments and formatting. And when reading and writing XML, using an option to preserve whitespace ensures the file remains readability.<br/>
Reference [PreserveWhitespace](https://learn.microsoft.com/en-us/dotnet/api/system.xml.xmldocument.preservewhitespace) for detailed information on that XmlDocument Property.<br/>

```powershell
# Create a namespace manager
$nsmgr = New-Object System.Xml.XmlNamespaceManager($xml.NameTable)
$nsmgr.AddNamespace("log4j", "http://logging.apache.org/")
```
```powershell
# Dynamically create a namespace manager
$nsmgr = New-Object System.Xml.XmlNamespaceManager($xml.NameTable)
foreach ($attribute in $xml.DocumentElement.Attributes) {
    $nsMgr.AddNamespace($attribute.LocalName, $attribute.Value)
}
```

### Why an XML Namespace Manager?
A namespace manager is necessary when working with XML documents that use namespaces, because it tells the XPath engine how to resolve the prefixes used in your query.

Typically an XML document is defined with namespaces, and namespaces are used to avoid naming conflicts between elements. from different XML vocabularies. Take a look at [w3schools XML Namespaces](https://www.w3schools.com/xml/xml_namespaces.asp) for reference.

Simpler XML documents may not be namespaced, and in such cases you don't need to think about namespaces. The first XML fragments examples at w3schools are not namespaced, but when those these fragments get added together, their attribute names will conflict and they will need to be namespaced to avoid the conflict.

My XML example contains one namespace, while more complex examples can have multiple. NB: The [foreach](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_foreach) method above captures namespace defined in the root node. If your document has other namespaces, that example wont capture them all.

Reference [Managing Namespaces in an XML Document](https://learn.microsoft.com/en-us/dotnet/standard/data/xml/managing-namespaces-in-an-xml-document)

### A demo of multiple options, successful and not.
```powershell
## Option 1: Fails as logger elements are unnamespaced, not prefixed
# Refs namespace prefix "log4j", node "logger", attribute filter "name='Logger1'" and $nsmgr.
$logger1 = $xml.SelectSingleNode("//log4j:logger[@name='Logger1']", $nsmgr)
$Logger2 = $xml.SelectSingleNode("//log4j:logger[@name='Logger2']", $nsmgr)

## Option 1b: Fails for the same reason
# Refs namespace prefix "log4j", parent node "configuration", child element "logger", and attribute  "name='Logger1'"
$logger1 = $xml.SelectSingleNode("//log4j:configuration/log4j:logger[@name='Logger1']", $nsmgr)
$Logger2 = $xml.SelectSingleNode("//log4j:configuration/log4j:logger[@name='Logger2']", $nsmgr)

## Option 1c: Succeeds
# Refs namespace prefix "log4j", root node "configuration", element "logger", and attribute "name='Logger1'"
$logger1 = $xml.SelectSingleNode("/log4j:configuration/logger[@name='Logger1']", $nsmgr)
$Logger2 = $xml.SelectSingleNode("/log4j:configuration/logger[@name='Logger2']", $nsmgr)

## Option 2: Succeeds with wildcards and ignoring namespaces
# Uses "/*[local-name()='configuration']" to match root element "configuration"
# Uses "/*[local-name()='logger']" to match child element "logger"
# Uses attribute filter "name='Logger1'"
# XmlNamespaceManager not needed because this ignores namespaces entirely
$logger1 = $xml.SelectSingleNode("/*[local-name()='configuration']/*[local-name()='logger'][@name='Logger1']")
$Logger2 = $xml.SelectSingleNode("/*[local-name()='configuration']/*[local-name()='logger'][@name='Logger2']")

## Option 3 - Success with a fully dynamic selection (may not suit your use case)
# 1. Using literal element name
$logger1 = $xml.SelectSingleNode("//logger[@name='Logger1']", $nsmgr)
# 2. Using wildcard for element name, ignores specific tag
$logger2 = $xml.SelectSingleNode("//*[@name='Logger2']", $nsmgr)
```

### With the node and its attribute(s) captured, update it
There are many ways, simplest is best, and there are many more ways to make this more complex.
```powershell
# Update level values
$logger1.level.value = "debug"
$logger2.level.value = "debug"

# Update level values
$logger1.level.Attributes['value'].Value = 'debug'
$logger2.level.Attributes['value'].Value = 'debug'

# Update level values
$logger1.SelectSingleNode('level').Attributes['value'].Value = 'debug'
$logger2.SelectSingleNode('level').Attributes['value'].Value = 'debug'
```

### With the attribute updated, save it.

The simplest of all does not produce a good result in the output file.

This simple `Save()` method uses PreserveWhitespace $true and will preserve whitespace, but it will remove comments and  the indentations that help with human readability (machines don't need the indentation).
```powershell
$xml.Save($outFile)
```
The better method is to take control over those details by using [XmlWriterSettings](https://learn.microsoft.com/en-us/dotnet/api/system.xml.xmlwritersettings) and [XmlWriter](https://learn.microsoft.com/en-us/dotnet/api/system.xml.xmlwriter)


### XmlWriterSettings
The `XmlWriterSettings` used below defines a settings configuration then used by `XmlWriter` when saving the document.

**UTF-8 without BOM** is an option I needed when not using it causes the file to be saved with encoding = UTF-8-BOM, and that was not desirable for my use case. Decide if your use case needs to handle that.

```powershell
# XmlWriterSettings
$settings = New-Object System.Xml.XmlWriterSettings
$settings.Encoding = [System.Text.UTF8Encoding]::new($false) # UTF-8 without BOM (Unicode byte order mark)
$settings.Indent = $true  # Indent elements on new lines

# XmlWriter
$writer = [System.Xml.XmlWriter]::Create($outFile, $settings)
$xml.Save($writer)
$writer.Close()   # closes this stream
$writer.Dispose() # releases the resources used
```

Here’s a simple diagram illustrating the XML DOM hierarchy for this example.
```text
<configuration>                      ← Parent element
│
├─ <logger name="Logger1">           ← Child element node
│    ├─ Attribute: name="Logger1"    ← Attribute node
│    ├─ Text nodes / inner content   ← Text nodes (if any)
│    └─ Child elements (nested)      ← Could be other elements inside logger
│
├─ <logger name="Logger2">           ← Another child element node
│
└─ <!-- Comment -->                  ← Comment node (also a child node)
```

### A more complex XML example (just for fun)

And don't forget LLM training is strong when it comes to XML, so chat with your favourite LLM on this.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!--
===================================================
 Observations about this XML:
 - Root is prefixed with log4j namespace
 - Child elements (logger, appender, root) are NOT 
   namespace-qualified (no prefix).
 - Each appender controls file rotation:
    - fileNamePattern
    - minIndex / maxIndex (backup count)
    - maxFileSize
 - Each logger has a <level> with @value controlling
   verbosity (debug, info, warn, error).
 - The mapping is: logger -> appender -> file.
 - White space + comments are used heavily, so any
   processing script should preserve them.
===================================================
-->

<log4j:configuration xmlns:log4j="http://logging.apache.org/">

    <!-- Example rolling file appender -->
    <appender name="App1" class="org.apache.log4j.rolling.RollingFileAppender">
        <rollingPolicy class="org.apache.log4j.rolling.FixedWindowRollingPolicy">
           <param name="fileNamePattern" value="D:\\Logs\\Anon\\App1%i.log"/>
           <param name="minIndex" value="0"/>
           <param name="maxIndex" value="5"/>
        </rollingPolicy>

        <triggeringPolicy class="org.apache.log4j.rolling.SizeBasedTriggeringPolicy">
            <param name="maxFileSize" value="50MB"/>
        </triggeringPolicy>

        <layout class="org.apache.log4j.PatternLayout">
            <param name="ConversionPattern" value="%d{dd/MM HH:mm:ss.SSS} [%-7t] %-5p %c - %m%n"/>
        </layout>

        <param name="append" value="true"/>
    </appender>
    
    <!-- Another appender, different rotation policy -->
    <appender name="App2" class="org.apache.log4j.rolling.RollingFileAppender">
        <rollingPolicy class="org.apache.log4j.rolling.FixedWindowRollingPolicy">
           <param name="fileNamePattern" value="D:\\Logs\\Anon\\App2%i.log"/>
           <param name="minIndex" value="0"/>
           <param name="maxIndex" value="1"/>
        </rollingPolicy>
        <triggeringPolicy class="org.apache.log4j.rolling.SizeBasedTriggeringPolicy">
            <param name="maxFileSize" value="10MB"/>
        </triggeringPolicy>
        <layout class="org.apache.log4j.PatternLayout">
            <param name="ConversionPattern" value="%d{dd/MM HH:mm:ss.SSS} [%-7t] %-5p %c - %m%n"/>
        </layout>
        <param name="append" value="true"/>
    </appender>
	
    <!-- Third appender, different format -->
    <appender name="App3" class="org.apache.log4j.rolling.RollingFileAppender">
        <rollingPolicy class="org.apache.log4j.rolling.FixedWindowRollingPolicy">
           <param name="fileNamePattern" value="D:\\Logs\\Anon\\App3%i.log"/>
           <param name="minIndex" value="0"/>
           <param name="maxIndex" value="5"/>
        </rollingPolicy>
        <triggeringPolicy class="org.apache.log4j.rolling.SizeBasedTriggeringPolicy">
            <param name="maxFileSize" value="50MB"/>
        </triggeringPolicy>
        <layout class="org.apache.log4j.PatternLayout"> 
            <param name="ConversionPattern" value="%d{dd/MM HH:mm:ss.SSS} [%-7t] - %m%n"/>
        </layout>
        <param name="append" value="true"/>
    </appender>

    <!-- Appender with slightly different timestamp pattern -->
    <appender name="App4" class="org.apache.log4j.rolling.RollingFileAppender"> 
        <rollingPolicy class="org.apache.log4j.rolling.FixedWindowRollingPolicy">
           <param name="fileNamePattern" value="D:\\Logs\\Anon\\App4%i.log"/>
           <param name="minIndex" value="0"/>
           <param name="maxIndex" value="5"/>
        </rollingPolicy>
        <triggeringPolicy class="org.apache.log4j.rolling.SizeBasedTriggeringPolicy">
            <param name="maxFileSize" value="50MB"/>
        </triggeringPolicy>
        <layout class="org.apache.log4j.PatternLayout">
            <param name="ConversionPattern" value="%d{dd/MM/yyyy-HH:mm:ss.SSS} [%t] - %m%n"/>
        </layout>
        <param name="Append" value="true"/>       
    </appender>
	
    <!-- Another appender, same as above -->
    <appender name="App5" class="org.apache.log4j.rolling.RollingFileAppender"> 
        <rollingPolicy class="org.apache.log4j.rolling.FixedWindowRollingPolicy">
           <param name="fileNamePattern" value="D:\\Logs\\Anon\\App5%i.log"/>
           <param name="minIndex" value="0"/>
           <param name="maxIndex" value="5"/>
        </rollingPolicy>
        <triggeringPolicy class="org.apache.log4j.rolling.SizeBasedTriggeringPolicy">
            <param name="maxFileSize" value="50MB"/>
        </triggeringPolicy>
        <layout class="org.apache.log4j.PatternLayout">
            <param name="ConversionPattern" value="%d{dd/MM/yyyy-HH:mm:ss.SSS} [%t] - %m%n"/>
        </layout>
        <param name="Append" value="true"/>       
    </appender>

    <!-- Logger definitions bind a name to an appender -->
    <logger name="Logger1">
        <level value="info"/>
        <appender-ref ref="App1"/>
    </logger>

    <logger name="Logger2">
        <level value="error"/>
        <appender-ref ref="App2"/>
    </logger>
    
    <logger name="Logger3">
        <level value="warn"/>
        <appender-ref ref="App3"/>
    </logger>
	
    <logger name="Logger4">
        <level value="info"/>
        <appender-ref ref="App5"/>
    </logger>
    
    <!-- Example with additivity flag -->
    <logger name="Logger1.Child" additivity="false">
        <level value="warn"/>
        <appender-ref ref="App1"/>
    </logger>

    <!-- Logger for message traffic -->
    <logger name="Logger5">
        <level value="debug"/>
        <appender-ref ref="App4"/>
    </logger>
    
    <!-- Root logger: default fallback -->
    <root> 
        <priority value="error"/>
    </root>     

</log4j:configuration>

```
