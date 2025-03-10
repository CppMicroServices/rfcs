- Start Date: 2024-05-15
- RFC PR: (in a subsequent commit to the PR, fill me with the PR's URL)
- CppMicroServices Issue: 

# Update LogService class in CppMicroServices

## Summary

The Log Service provides a general purpose message logger for the OSGi framework. It consists of several services: a service for obtaining Loggers to log information and other services for retrieving current or previously recorded log information.
This task is to support latest OSGi features in LogService class (according to OSGi spec 8- https://docs.osgi.org/specification/osgi.core/8.0.0/service.log.html)

## Motivation

The current CppMicroServices LogService API and implementation that we ship with CppMicroServices does not confirm to the latest LogService OSGi spec.
The current 'LogService' class diagram looks like-

![Old LogService class](https://github.com/architapruthi/rfcs/blob/master/logService_old.png)

Everything within red is not supported.

Now, according to the latest OSGi spec, the 'LogService' looks like-

![New LogService class](https://github.com/architapruthi/rfcs/blob/master/logService_new.png)

The latest spec introduces LoggerFactory service interface that allows a bundle to obtain a Logger. And the Logger in turn allows a bundle to log information, including a message, a level, an exception, and a ServiceReference object.
Hence, the task is to update the implementation of open-source LogService and stay current with the latest OSGi standard.

## Detailed design

We will see how different interfaces will look like according to new OSGi spec-

The namespace for all these interfaces will be "cppmicroservices::logservice"

1. The LoggerFactory interface :

> The LoggerFactory service can be used to obtain two types of Logger objects: Logger and FormatterLogger. But, currently since there is no use-case for FormatterLogger, we will not be providing this interface.
> The Logger object uses SLF4J-style ("{}") place holders for message formatting.

```c++
namespace cppmicroservices
{
    namespace logservice
    {
        class LoggerFactory
        {
            public:
                virtual ~LoggerFactory() = default;
 
                virtual std::shared_ptr<Logger> getLogger(const std::string& name) const = 0;
 
                virtual std::shared_ptr<LoggerType> getLogger(Bundle bundle, const std::string& name) const = 0;
 
        };
    }
}
```

2. The Logger interface :

> Provides methods for bundles to write messages to the log using SLF4J-style format strings. 
> Messages can be formatted by the Logger once the Logger determines the log level is enabled. Use a left curly bracket ('{' \u007B) followed by a right curly bracket ('}' \u007D) as a place holder for an argument: "{}"
> The Logger interface allows the bundle developer to:

  Specify a message, message parameters, and an exception to be logged.
  Specify the Service associated with the message being logged.

> The Logger interface defines several methods for each of the defined Log-levels :

```c++
enum class LogLevel
{
    AUDIT , //This log level is used for information that must always be logged.
 
    ERROR , //This log level is used for information about an error situation.
 
    WARN  , //This log level is used for information about a failure or unwanted situation that is not blocking.
 
    INFO  , //This log level is used for information about normal operation.
 
    DEBUG , //This log level is used for detailed output for debugging operations.
 
    TRACE  //This log level is used for large volume of output for tracing operations.
};
```

> The 'Logger' interface, which defines various methods for logging, including 'audit', 'debug', 'error', 'info', 'trace', 'warn', will appear as follows -

```c++
class Logger
{
    public:

        virtual ~Logger() = default;

        virtual void audit(const std::string& message) = 0;
        virtual void audit(const std::string& format, const std::string& arg) = 0;
        virtual void audit(const std::string& format, const std::string& arg1, const std::string& arg2) = 0;
        virtual void audit(std::string const& message, const std::exception_ptr ex) = 0;
        virtual void audit(std::string const& message, ServiceReferenceBase const& sr) = 0;
        virtual void audit(std::string const& message, ServiceReferenceBase const& sr, const std::exception_ptr ex) = 0;

        virtual void debug(const std::string& message) = 0;
        virtual void debug(const std::string& format, const std::string& arg) = 0;
        virtual void debug(const std::string& format, const std::string& arg1, const std::string& arg2) = 0;
        virtual void debug(std::string const& message, const std::exception_ptr ex) = 0;
        virtual void debug(std::string const& message, ServiceReferenceBase const& sr) = 0;
        virtual void debug(std::string const& message, ServiceReferenceBase const& sr, const std::exception_ptr ex) = 0;

        ......And so on, for other log methods....
}
```
> Internally, we will be using 'spdlog' library which provides formatting capabilities for logging messages. It supports various format specifiers, such as {} for replacement fields, allowing you to format log messages dynamically with different data types.

> In this task, we will not be supporting template overloads of the logging methods.

3. LogService interface:

> The members of the LogService interface are deprecated. We will continue to provide support for deprecated functionality in the short term to minimize disruption for existing users. As and when OSGi spec will state that the support is not provided for LogService, we will also remove the support of it.

> Its 'log' methods can still be used by bundles. These log methods are now specified to log to the Logger with the logger name "LogService.<bsn>, where <bsn> is the Bundle Symbolic Name of the bundle which obtained the LogService object. This allows legacy logging to be configured as specified above.

> Furthermore, the integer log level values used with the log methods are mapped to the new LogLevels as follows:

  LOG_ERROR is mapped to ERROR.
  LOG_WARNING is mapped to WARN.
  LOG_INFO is mapped to INFO.
  LOG_DEBUG is mapped to DEBUG.
  Any other value is mapped to TRACE.

> Hence, to maintain backward compatibility,  LogService class will look like -

```c++
class LogService : public LoggerFactory
{
    private:
        Logger logger; // Create Logger instance using LoggerFactory's 'getLogger()' method

    public:
        void log(int level, const std::string& message) {
            // Map log level to corresponding LogLevels
            std::string logLevel;
            switch (level) {
                case LOG_ERROR:
                    logger.error(message);
                    break;
                case LOG_WARNING:
                    logger.warn(message);
                    break;
                case LOG_INFO:
                    logger.info(message);
                    break;
                case LOG_DEBUG:
                    logger.debug(message);
                    break;
                default:
                    logger.trace(message);
                    break;
            }
        }
};
```

>> Out Of Scope:

1. The FormatterLogger interface: This is not a requirement as of now since there are no use-cases for using this style of formatting today. Users can use 'Logger' interface which supports SLF4J style of formatting.
2. A Logger Admin service which allows for the configuration of Loggers is not handled in this part of the task. Please see more details in latest OSGi spec for Logger.
3. 'LoggerConsumer' -  An operation that accepts a 'Logger' argument and produces no result.  Sometimes message parameters can be expensive to compute, so avoiding computation is important if the log level is not effective. This can be done using a LoggerConsumer. This is convenient as a lambda expression.


## How we teach this

The API documentation and OSGi spec for LogService should suffice to educate clients on how to use this feature.

## Drawbacks

To ensure backward compatibility of LogService and prevent any disruptions to existing code,
> LogService will inherit from LoggerFactory, allowing the LogService interface to include a method for obtaining the logger using the 'getLogger' method in addition to its various logging methods.

> Internally, we will be using 'LoggerFactory' and 'Logger' classes to obtain logger and call logging functions, respectively.

The drawback here is: If we inherit LogService from LoggerFactory, then all other services which use LogService will have to implement 'getLogger' functionality. This way, we will be modifying existing Cppms framework.

But, this design is in adherence to OSGi spec.

## Alternatives

The alternative is to not inherit LogService from LoggerFactory, instead, create a separate interface which will inherit from both LogService and LoggerFactory interfaces. Hence, it will have all the methods  from LogService as well as LoggerFactory.

But, this design does not conform to OSGi spec.
