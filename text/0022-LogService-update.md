- Start Date: 2024-05-15
- RFC PR: https://github.com/CppMicroServices/rfcs/pull/21
- CppMicroServices Issue: (none). Implemented in https://github.com/CppMicroServices/CppMicroServices/pull/1009 (commit `070fd93e`).

# Update LogService class in CppMicroServices

## Summary

The Log Service provides a general-purpose message logger for the OSGi framework. It consists of several services: one for obtaining Loggers to log information, and others for retrieving current or previously recorded log information.
This task updates the LogService class to support the latest OSGi features (OSGi spec 8: https://docs.osgi.org/specification/osgi.core/8.0.0/service.log.html).

## Motivation

The current CppMicroServices LogService API and implementation that we ship with CppMicroServices does not conform to the latest LogService OSGi spec.
The current 'LogService' class diagram looks like this:

![Old LogService class](https://github.com/architapruthi/rfcs/blob/master/logService_old.png)

Everything within red is not supported.

According to the latest OSGi spec, the 'LogService' looks like this:

![New LogService class](https://github.com/architapruthi/rfcs/blob/master/logService_new.png)

The latest spec introduces the LoggerFactory service interface, which allows a bundle to obtain a Logger. The Logger in turn allows a bundle to log information, including a message, a level, an exception, and a ServiceReference object.
The task is therefore to update the open-source LogService implementation so that it stays current with the latest OSGi standard.

## Detailed design

This section describes the interfaces as implemented. The public headers are in `compendium/LogService/include/cppmicroservices/logservice/`. The implementation is in `compendium/LogServiceImpl/src/`.

The namespace for all these interfaces is `cppmicroservices::logservice`.

1. The LoggerFactory interface (`LoggerFactory.hpp`):

> The LoggerFactory service can be used to obtain two types of Logger objects: Logger and FormatterLogger. There is currently no use case for FormatterLogger, so that interface is not provided.
> The Logger object uses SLF4J-style ("{}") placeholders for message formatting.

```c++
namespace cppmicroservices::logservice
{
    class LoggerFactory
    {
      public:
        inline static const std::string ROOT_LOGGER_NAME = "ROOT";

        virtual ~LoggerFactory() = default;

        [[nodiscard]] virtual std::shared_ptr<Logger> getLogger(std::string const& name = ROOT_LOGGER_NAME) const = 0;

        [[nodiscard]] virtual std::shared_ptr<Logger> getLogger(const cppmicroservices::Bundle& bundle, std::string const& name = ROOT_LOGGER_NAME) const = 0;
    };
}
```

> Logger naming in the implementation (`LoggerImpl.cpp:43-58`):
> - `getLogger(name)` returns a logger named `name`.
> - `getLogger(bundle, name)` returns a logger named `<bundle symbolic name>.<name>`.
>
> `ROOT_LOGGER_NAME` is only the default value of `name`. There is no shared root logger: `getLogger()` returns a logger named `ROOT`, and `getLogger(bundle)` returns one named `<bundle symbolic name>.ROOT`.
>
> Each `getLogger` call creates a new `Logger` (`LoggerFactoryImpl.cpp`). Loggers are not cached or registered, so two calls with the same name return two independent logger objects.
>
> > **Note:** the `LoggerFactory` API documentation says that "the root logger is returned" for `ROOT_LOGGER_NAME`, which the implementation does not do. This appears to be a bug; see [CppMicroServices#1308](https://github.com/CppMicroServices/CppMicroServices/issues/1308).

2. The Logger interface (`Logger.hpp`):

> Provides methods for bundles to write messages to the log using SLF4J-style format strings. Use a left curly bracket ('{' \u007B) followed by a right curly bracket ('}' \u007D) as a placeholder for an argument: "{}".
> The Logger interface allows the bundle developer to:

  Specify a message, message parameters, and an exception to be logged.
  Specify the Service associated with the message being logged.

> The Logger interface defines a set of methods for each of the defined log levels:

```c++
enum class LogLevel
{
    Audit, // This log level is used for information that must always be logged.
    Error, // This log level is used for information about an error situation.
    Warn,  // This log level is used for information about a failure or unwanted situation that is not blocking.
    Info,  // This log level is used for information about normal operation.
    Debug, // This log level is used for detailed output for debugging operations.
    Trace  // This log level is used for large volume of output for tracing operations.
};
```

> `LogLevel` is declared in `Logger.hpp` for documentation, but no API takes or returns it. `Logger` has no `isXxxEnabled()` or `getName()` methods.

> The `Logger` interface defines six overloads for each of `audit`, `debug`, `error`, `info`, `trace` and `warn`, which is 36 methods in all. Format arguments are supported only on the message-only overloads:

```c++
class Logger
{
  public:
    virtual ~Logger() = default;

    virtual void audit(std::string const& message) = 0;
    virtual void audit(std::string const& format, std::string const& arg) = 0;
    virtual void audit(std::string const& format, std::string const& arg1, std::string const& arg2) = 0;
    virtual void audit(std::string const& message, const std::exception_ptr ex) = 0;
    virtual void audit(std::string const& message, ServiceReferenceBase const& sr) = 0;
    virtual void audit(std::string const& message, ServiceReferenceBase const& sr, const std::exception_ptr ex) = 0;

    virtual void debug(std::string const& message) = 0;
    virtual void debug(std::string const& format, std::string const& arg) = 0;
    virtual void debug(std::string const& format, std::string const& arg1, std::string const& arg2) = 0;
    virtual void debug(std::string const& message, const std::exception_ptr ex) = 0;
    virtual void debug(std::string const& message, ServiceReferenceBase const& sr) = 0;
    virtual void debug(std::string const& message, ServiceReferenceBase const& sr, const std::exception_ptr ex) = 0;

    // ... and the same six overloads for error, info, trace and warn.
};
```

> Internally, the implementation uses the 'spdlog' library, which provides formatting for log messages. The format and arguments are passed straight to spdlog, so `{}` replacement fields work as described above.

> Template overloads of the logging methods are not supported.

> Current behaviour of the implementation (`LoggerImpl.cpp`):
> - Each logger writes to its own colored stdout sink (`spdlog::sinks::stdout_color_sink_mt`), using the pattern `[%T] [%P:%t] %n (%^%l%$): %v`. No other sink can be configured through the public API. `LoggerImpl::AddSink` and `LogServiceImpl::AddSink` exist only in the implementation, and are used by tests.
> - An exception is appended to the message as `"\nException logged: <type> : <what()>"`, or `"\nException logged: none"` for a null `exception_ptr`. A ServiceReference is appended as `"\nServiceReference: <sr>"`.
> - **No level filtering.** Each logging method first sets the underlying spdlog logger's level to its own level, then logs. For example, `debug()` calls `set_level(debug)` and then `debug(...)`. Every message is therefore emitted, and nothing can change that. Because the level lives on a logger instance that callers may share across threads, concurrent calls at different levels can make spdlog drop a message.
> - **Audit is logged as trace.** `audit(...)` logs through spdlog's `trace` level, so audit records are labelled `trace` in the output.
>
> > **Note:** this appears to be a bug; see [CppMicroServices#1304](https://github.com/CppMicroServices/CppMicroServices/issues/1304).

3. LogService interface (`LogService.hpp`):

> The `LogService` `Log` methods are kept for backward compatibility. They are not marked `[[deprecated]]`.

> The implementation logs these calls to a Logger named `LogService.<bsn>`, where `<bsn>` is the symbolic name of the LogService implementation bundle (`log_service`). The activator registers a single `LogServiceImpl` instance, not a `ServiceFactory` (`LogServiceImpl/src/Activator.cpp:10-15`). Every bundle therefore logs through the same `LogService.log_service` logger, whichever bundle obtained the service.
>
> > **Note:** this appears to be a bug; OSGi specifies the logger name as `LogService.<bsn>` of the bundle that obtained the LogService. See [CppMicroServices#1305](https://github.com/CppMicroServices/CppMicroServices/issues/1305).

> The `SeverityLevel` values used with the `Log` methods (`LOG_ERROR = 1`, `LOG_WARNING = 2`, `LOG_INFO = 3`, `LOG_DEBUG = 4`) are mapped to the new log levels as follows:

  LOG_ERROR is mapped to error.
  LOG_WARNING is mapped to warn.
  LOG_INFO is mapped to info.
  LOG_DEBUG is mapped to debug.
  Any other value is mapped to trace.

> `LogService` is a pure interface that inherits `LoggerFactory`:

```c++
enum class SeverityLevel : uint8_t
{
    LOG_ERROR = 1,
    LOG_WARNING = 2,
    LOG_INFO = 3,
    LOG_DEBUG = 4
};

class LogService : public LoggerFactory
{
  public:
    virtual ~LogService() = default;

    virtual void Log(SeverityLevel level, std::string const& message) = 0;
    virtual void Log(SeverityLevel level, std::string const& message, const std::exception_ptr ex) = 0;
    virtual void Log(ServiceReferenceBase const& sr, SeverityLevel level, std::string const& message) = 0;
    virtual void Log(ServiceReferenceBase const& sr,
                     SeverityLevel level,
                     std::string const& message,
                     const std::exception_ptr ex)
        = 0;
};
```

> The implementation, `LogServiceImpl`, holds a `std::shared_ptr<Logger>` obtained from `getLogger(loggerName)` in its constructor. Each `Log` overload forwards to `LogServiceImpl::logImpl`, which switches on `SeverityLevel` and calls the matching `Logger` method (`LogServiceImpl.hpp`). `LogServiceImpl::getLogger` creates a new `LoggerFactoryImpl` on each call and delegates to it. The service is registered under both the `LogService` and `LoggerFactory` interfaces.

>> Out Of Scope:

1. The FormatterLogger interface: there is no requirement for it, since no current use cases need this style of formatting. Users can use the 'Logger' interface, which supports SLF4J-style formatting.
2. A Logger Admin service, which would allow Loggers to be configured, is not part of this task. See the latest OSGi spec for Logger for details.
3. 'LoggerConsumer': an operation that accepts a 'Logger' argument and produces no result. Message parameters can be expensive to compute, so it helps to skip computing them when the log level is not effective. A LoggerConsumer does this and is convenient to write as a lambda expression.

None of these three is implemented.

## How we teach this

The API documentation and the OSGi spec for LogService should be enough to show clients how to use this feature. The API reference (`compendium/LogService/doc/api/LogService.rst`) is generated from the Doxygen comments in the public headers.

Tests are in `compendium/LogServiceImpl/test/`: `TestLoggerFactory.cpp` (`ProperLoggerUsage`, `InvalidLoggerUsage`, `ThreadSafety`) and `TestLogService.cpp`. No test currently covers `getLogger(bundle, name)` or the `LogService.<bsn>` logger name.

## Drawbacks

To keep LogService backward compatible and avoid disrupting existing code:
> LogService inherits from LoggerFactory. The LogService interface therefore includes `getLogger` for obtaining a logger, in addition to its `Log` methods.

> Internally, the 'LoggerFactory' and 'Logger' classes are used to obtain loggers and to call the logging functions.

The drawback is that, because LogService inherits from LoggerFactory, every other class that implements LogService must also implement 'getLogger'. This changes existing CppMicroServices code. Declarative Services (`SCRLogger`) and Configuration Admin (`CMLogger`) implement `getLogger` by forwarding to the tracked LogService. When no LogService is available, they return `nullptr` (`compendium/DeclarativeServices/src/SCRLogger.cpp:128-147`, `compendium/ConfigurationAdmin/src/CMLogger.cpp:132-151`).

But, this design is in adherence to OSGi spec.

## Alternatives

The alternative is not to have LogService inherit from LoggerFactory. Instead, a separate interface would inherit from both LogService and LoggerFactory, and so have all the methods of both.

But, this design does not conform to OSGi spec.
