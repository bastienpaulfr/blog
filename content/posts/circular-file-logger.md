---
title: "How to create a circlular file logger with Timber"
date: 2019-03-01T15:04:06+02:00
draft: false
---

In some applications, I need to store my logs in a file aside of traditional logcat. For this, I am making use of [Timber](https://github.com/JakeWharton/timber) library. Because I don’t want to make my device full of logs, I wanted to use circular log files so that I can control the maximum amount of bytes taken by log data. To achieve this, I will use java Logger API to implement a new `Timber.Tree`. I also want some feature like log formatting and filtering.

All of this is implemented by [Treessence](https://github.com/bastienpaulfr/Treessence) library.

## Log filtering

To implement filtering an interface is defined :

```java
public interface Filter {

    /**
     * @param priority Log priority.
     * @param tag      Tag for log.
     * @param message  Formatted log message.
     * @param t        Accompanying exceptions.
     * @return {@code true} if the log should be skipped, otherwise {@code false}.
     * @see timber.log.Timber.Tree#log(int, String, String, Throwable)
     */
    boolean skipLog(int priority, String tag, String message, Throwable t);

    boolean isLoggable(int priority, String tag);
}
```

Priority filtering is provided by an implementation of this interface

```java
public class PriorityFilter implements Filter {

    private final int minPriority;

    public PriorityFilter(int minPriority) {
        this.minPriority = minPriority;
    }

    @Override
    public boolean skipLog(int priority, String tag, String message, Throwable t) {
        return priority < minPriority;
    }

    @Override
    public boolean isLoggable(int priority, String tag) {
        return priority >= minPriority;
    }

    public int getMinPriority() {
        return minPriority;
    }
}
```

We can now create our base class extending `Timber.DebugTree`

```java
public class PriorityTree extends Timber.DebugTree {

    private final PriorityFilter priorityFilter;
    private Filter filter = NoFilter.INSTANCE;

    /**
     * @param priority priority from witch log will be logged
     */
    public PriorityTree(int priority) {
        this.priorityFilter = new PriorityFilter(priority);
    }

    /**
     * Add additional {@link Filter}
     *
     * @param f Filter
     * @return itself
     */
    public PriorityTree withFilter(@NotNull Filter f) {
        this.filter = f;
        return this;
    }

    @Override
    protected boolean isLoggable(int priority) {
        return isLoggable("", priority);
    }

    @Override
    public boolean isLoggable(String tag, int priority) {
        return priorityFilter.isLoggable(priority, tag) && filter.isLoggable(priority, tag);
    }

    public PriorityFilter getPriorityFilter() {
        return priorityFilter;
    }

    public Filter getFilter() {
        return filter;
    }

    /**
     * Use the additional filter to determine if this log needs to be skipped
     *
     * @param priority Log priority
     * @param tag      Log tag
     * @param message  Log message
     * @param t        Log throwable
     * @return true if needed to be skipped or false
     */
    protected boolean skipLog(int priority, String tag, @NotNull String message, Throwable t) {
        return filter.skipLog(priority, tag, message, t);
    }
}
```

This class can filter on two parameters :

- First parameter is obviously log priority. This is done thanks to PriorityFilter instance.
- Second parameter is an additional Filter instance that can be provided by caller.

## Log formatting

Log formatting is obtained thanks to a Formatter class whose interface is defined as follow

```java
public interface Formatter {

    String format(int priority, String tag, String message);
}
```

Each formatter can display log to a defined format. For instance, logcat format is `"MM-dd HH:mm:ss:SSS {priority}/{tag}({thread id}) : {message}\n"`. Another format would be `"{tag} : {message}"`.

Because we want to log in a file what we get in logcat, then we need to implement a logcat formatter

```java
public class LogcatFormatter implements Formatter {

    public static final LogcatFormatter INSTANCE = new LogcatFormatter();
    private static final String SEP = " ";

    private final HashMap<Integer, String> prioPrefixes = new HashMap<>();

    private LogcatFormatter() {
        prioPrefixes.put(Log.VERBOSE, "V/");
        prioPrefixes.put(Log.DEBUG, "D/");
        prioPrefixes.put(Log.INFO, "I/");
        prioPrefixes.put(Log.WARN, "W/");
        prioPrefixes.put(Log.ERROR, "E/");
        prioPrefixes.put(Log.ASSERT, "WTF/");
    }

    @Override
    public String format(int priority, String tag, @NotNull String message) {
        String prio = prioPrefixes.get(priority);
        if (prio == null) {
            prio = "";
        }
        return TimeUtils.timestampToDate(System.currentTimeMillis(), "MM-dd HH:mm:ss:SSS")
               + SEP
               + prio
               + (tag == null ? "" : tag)
               + "(" + Thread.currentThread().getId() + ") :"
               + SEP
               + message
               + "\n";
    }
}
```

Priority class can then be extended to add format functionality

```java
/**
 * Base class to format logs
 */
public class FormatterPriorityTree extends PriorityTree {
    private Formatter formatter = getDefaultFormatter();

    public FormatterPriorityTree(int priority) {
        super(priority);
    }

    /**
     * Set {@link Formatter}
     *
     * @param f formatter
     * @return itself
     */
    public FormatterPriorityTree withFormatter(Formatter f) {
        this.formatter = f;
        return this;
    }

    /**
     * Use its formatter to format log
     *
     * @param priority Priority
     * @param tag      Tag
     * @param message  Message
     * @return Formatted log
     */
    protected String format(int priority, String tag, @NotNull String message) {
        return formatter.format(priority, tag, message);
    }

    /**
     * @return Default log {@link Formatter}
     */
    protected Formatter getDefaultFormatter() {
        return NoTagFormatter.INSTANCE;
    }

    @Override
    protected void log(int priority, String tag, @NotNull String message, Throwable t) {
        super.log(priority, tag, format(priority, tag, message), t);
    }
}
```

## File logging

We have seen how to filter and format logs. We can now start logging in file.

For this we need a `java.util.logging.Logger` instance. It will be used in conjunction with `java.util.logging.FileHandler` that do actual file logging. We will see how to create a Logger instance later.

```java
public class FileLoggerTree extends FormatterPriorityTree {
    private final Logger logger;

    private FileLoggerTree(int priority,
                           Logger logger) {
        super(priority);
        this.logger = logger;
    }
}
```

To activate logcat formatting by default, `getDefaultFormatter()` method is overridden


```java
@Override
protected fr.bipi.tressence.common.formatter.Formatter getDefaultFormatter() {
    return LogcatFormatter.INSTANCE;
}
```

We need to convert logcat level to `java.util.logging.Level`

```java
private Level fromPriorityToLevel(int priority) {
    switch (priority) {
        case Log.VERBOSE:
            return Level.FINER;
        case Log.DEBUG:
            return Level.FINE;
        case Log.INFO:
            return Level.INFO;
        case Log.WARN:
            return Level.WARNING;
        case Log.ERROR:
            return Level.SEVERE;
        case Log.ASSERT:
            return Level.SEVERE;
        default:
            return Level.FINEST;
    }
}
```

Logging is done by this method

```java
@Override
protected void log(int priority, String tag, @NotNull String message, Throwable t) {
    if (skipLog(priority, tag, message, t)) {
        return;
    }

    logger.log(fromPriorityToLevel(priority), format(priority, tag, message));
    if (t != null) {
        logger.log(fromPriorityToLevel(priority), "", t);
    }
}
```

It is logging in using `java.utils.logging` API with log level conversion and logcat formatting

We haven’t seen how to provide the right logger. Let see how to configure it.

A builder class is defined to create a FileLoggerTree instance. This builder contains some default:

```java
public static class Builder {
    // 1 mb byte of data
    private static final int SIZE_LIMIT = 1048576;
    // Max 3 files for circular logging
    private static final int NB_FILE_LIMIT = 3;

    // Base filename.
    // log index will be appended so actual file name will be
    // "log.0" or "log.1"
    // To parametrize where index is put, "%g" can be placed
    // in file name. For instance "log%g.logcat" will give
    // "log0.logcat", "log1.logcat" and so on
    private String fileName = "log";
    // Directory where files are stored
    private String dir = "";
    // Min priority to log from
    private int priority = Log.INFO;
    private int sizeLimit = SIZE_LIMIT;
    private int fileLimit = NB_FILE_LIMIT;
    // append log to already existing log file
    private boolean appendToFile = true;

...
}
```

`java.util.logging.Logger` are created and managed by `java.util.logging.LogManager`. To bypass this a simple static class is used


```java
/**
 * Custom logger class that has no references to LogManager
 */
private static class MyLogger extends Logger {

    /**
     * Constructs a {@code Logger} object with the supplied name and resource
     * bundle name; {@code notifyParentHandlers} is set to {@code true}.
     * <p/>
* Notice : Loggers use a naming hierarchy. Thus "z.x.y" is a child of "z.x".
     *
     * @param name the name of this logger, may be {@code null} for anonymous
     *             loggers.
     */
    MyLogger(String name) {
        super(name, null);
    }

    public static Logger getLogger(String name) {
        return new MyLogger(name);
    }
}
```

Creation of `java.util.logging.Logger` can start

```java
public FileLoggerTree build() throws IOException {
  // Log file path
  String path = FileUtils.combinePath(dir, fileName);
  // File handler that is performing file logging
  FileHandler fileHandler;
  // Our custom logger
  Logger logger = MyLogger.getLogger(TAG);
  // We force level to ALL because priority filtering is
  // done by our Tree implementation
  logger.setLevel(Level.ALL);
  // File handler can now be created
  fileHandler = new FileHandler(path, sizeLimit, fileLimit,  appendToFile);
  // Formating is done by our Tree implementation
  fileHandler.setFormatter(new NoFormatter());
  // Configure java Logger
  logger.addHandler(fileHandler);
  // finally we got here !
  return new FileLoggerTree(priority, logger);
}
```

Full code of `FileLoggerTree` is [here](https://github.com/bastienpaulfr/Treessence/blob/master/treessence/src/main/java/fr/bipi/tressence/file/FileLoggerTree.java)

This tree can then be planted like this

```java
FileLoggerTree fileTree = FileLoggerTree.Builder()
        .withFileName("log%g.logcat")
        .withMinPriority(Log.VERBOSE)
        .build()
Timber.plant(fileTree)
```

Thanks for reading this. Full source code is available [here](https://github.com/bastienpaulfr/Treessence)
