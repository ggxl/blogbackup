---
title: log4j日志扩展实现log文件同时按日期和大小切分
no_word_count: false
tags: [技术,后端,日志]
---


### log4j的几个核心类

- Logger用于对日志记录行为的抽象，提供记录不同级别日志的接口
- Level对日志级别的抽象
- Appender是对记录日志形式的抽象
- Layout是对日志行格式的抽象
- LoggingEvent是对一次日志记录过程中所能取到信息的抽象
- LoggerRepository是Logger实例的容器
- ObjectRender是对日志实例的解析接口，它们主要提供了一种扩展支持




### 平时常用的两个Appender类
	
-  DailyRollingFileAppender
-  RollingFileAppender

> DailyRollingFileAppender:按照日期格式切分
> RollingFileAppender：按照文件大小切分



### 两个关键方法

![](rollOver&subAppend两个关键方法.png '两个关键方法') 

### 分别看下代码

##### DailyRollingFileAppender代码

```java
/**
     Rollover the current file to a new file.
  */
  void rollOver() throws IOException {

    /* Compute filename, but only if datePattern is specified */
    if (datePattern == null) {
      errorHandler.error("Missing DatePattern option in rollOver().");
      return;
    }
	//假如format:yyyy-MM-dd ,当前时间为2018-07-26 

    String datedFilename = fileName+sdf.format(now);

    // It is too early to roll over because we are still within the
    // bounds of the current interval. Rollover will occur once the
    // next interval is reached.
	//scheduledFilename = fileName+sdf.format(new Date(file.lastModified()))
	//判断是否要切文件 ，scheduledFilename=filename.2018-07-26.log 在27日时不相等
    if (scheduledFilename.equals(datedFilename)) {
      return;
    }

    // close current file, and rename it to datedFilename
    this.closeFile();

    File target  = new File(scheduledFilename);//filename.2018-07-26.log
    if (target.exists()) {
      target.delete();
    }

    File file = new File(fileName);
    boolean result = file.renameTo(target);//将当前filename.log文件重命名filename.2018-07-26.log
    if(result) {
      LogLog.debug(fileName +" -> "+ scheduledFilename);
    } else {
      LogLog.error("Failed to rename ["+fileName+"] to ["+scheduledFilename+"].");
    }

    try {
      // This will also close the file. This is OK since multiple
      // close operations are safe.
      this.setFile(fileName, true, this.bufferedIO, this.bufferSize);
    }
    catch(IOException e) {
      errorHandler.error("setFile("+fileName+", true) call failed.");
    }
    scheduledFilename = datedFilename;
  }

  /**
   * This method differentiates DailyRollingFileAppender from its
   * super class.
   *
   * <p>Before actually logging, this method will check whether it is
   * time to do a rollover. If it is, it will schedule the next
   * rollover time and then rollover.
   * */
  protected void subAppend(LoggingEvent event) {
    long n = System.currentTimeMillis();
	//判断当前是否该切文件
    if (n >= nextCheck) {
      now.setTime(n);
      nextCheck = rc.getNextCheckMillis(now);//根据配置文件中配置的格式计算下一个切换文件时间
      try {
	     rollOver();
      }
      catch(IOException ioe) {
          if (ioe instanceof InterruptedIOException) {
              Thread.currentThread().interrupt();
          }
	      LogLog.error("rollOver() failed.", ioe);
      }
    }
    super.subAppend(event);
   }
}

```
##### RollingFileAppender代码

###### 首先了解下在RollingFileAppender中定义了三个属性
```java
 /**
     The default maximum file size is 10MB.
  */
  protected long maxFileSize = 10*1024*1024;

  /**
     There is one backup file by default.
   */
  protected int  maxBackupIndex  = 1;

  private long nextRollover = 0;
```
###### maxFileSize,maxBackupIndex属性的get set 以及在判断日志大小时需要用到的 qw 

```
 public void setMaxBackupIndex(int maxBackups) {
        this.maxBackupIndex = maxBackups;
    }
    public void setMaximumFileSize(long maxFileSize) {
        this.maxFileSize = maxFileSize;
    }
    public void setMaxFileSize(String value) {
        maxFileSize = OptionConverter.toFileSize(value, maxFileSize + 1);
    }
    protected void setQWForFiles(Writer writer) {
        this.qw = new CountingQuietWriter(writer, errorHandler);
    }
    public int getMaxBackupIndex() { return maxBackupIndex;}
    public long getMaximumFileSize() { return maxFileSize; }
```


```java

/**
     Implements the usual roll over behaviour.

     <p>If <code>MaxBackupIndex</code> is positive, then files
     {<code>File.1</code>, ..., <code>File.MaxBackupIndex -1</code>}
     are renamed to {<code>File.2</code>, ...,
     <code>File.MaxBackupIndex</code>}. Moreover, <code>File</code> is
     renamed <code>File.1</code> and closed. A new <code>File</code> is
     created to receive further log output.

     <p>If <code>MaxBackupIndex</code> is equal to zero, then the
     <code>File</code> is truncated with no backup files created.

   */
  public // synchronization not necessary since doAppend is alreasy synched
  void rollOver() {
    File target;
    File file;

    if (qw != null) {
        long size = ((CountingQuietWriter) qw).getCount();
        LogLog.debug("rolling over count=" + size);
        //   if operation fails, do not roll again until
        //      maxFileSize more bytes are written
        nextRollover = size + maxFileSize;
    }
    LogLog.debug("maxBackupIndex="+maxBackupIndex);

    boolean renameSucceeded = true;
    // If maxBackups <= 0, then there is no file renaming to be done.
    if(maxBackupIndex > 0) {
      // Delete the oldest file, to keep Windows happy.
      file = new File(fileName + '.' + maxBackupIndex);
      if (file.exists())
       renameSucceeded = file.delete();

      // Map {(maxBackupIndex - 1), ..., 2, 1} to {maxBackupIndex, ..., 3, 2}
      for (int i = maxBackupIndex - 1; i >= 1 && renameSucceeded; i--) {
	file = new File(fileName + "." + i);
	if (file.exists()) {//按照配置的最大编号递减循环，判断文件是否存在，存在的文件名编号修改为 编号+1 最终编号为1的变为2 空出编号1
	  target = new File(fileName + '.' + (i + 1));
	  LogLog.debug("Renaming file " + file + " to " + target);
	  renameSucceeded = file.renameTo(target);
	}
      }

    if(renameSucceeded) {
      // Rename fileName to fileName.1
      target = new File(fileName + "." + 1); 

      this.closeFile(); // keep windows happy.

      file = new File(fileName);
      LogLog.debug("Renaming file " + file + " to " + target);
      renameSucceeded = file.renameTo(target); //将fileName 重命名为 fileName.1
      //
      //   if file rename failed, reopen file with append = true
      //
      if (!renameSucceeded) {
          try {
            this.setFile(fileName, true, bufferedIO, bufferSize);
          }
          catch(IOException e) {
              if (e instanceof InterruptedIOException) {
                  Thread.currentThread().interrupt();
              }
              LogLog.error("setFile("+fileName+", true) call failed.", e);
          }
      }
    }
    }

    //
    //   if all renames were successful, then
    //
    if (renameSucceeded) {
    try {
      // This will also close the file. This is OK since multiple
      // close operations are safe.
      this.setFile(fileName, false, bufferedIO, bufferSize);
      nextRollover = 0;
    }
    catch(IOException e) {
        if (e instanceof InterruptedIOException) {
            Thread.currentThread().interrupt();
        }
        LogLog.error("setFile("+fileName+", false) call failed.", e);
    }
    }
  }


/**
     This method differentiates RollingFileAppender from its super
     class.

     @since 0.9.0
  */
  protected
  void subAppend(LoggingEvent event) {
    super.subAppend(event);
    if(fileName != null && qw != null) {
        long size = ((CountingQuietWriter) qw).getCount();
        if (size >= maxFileSize && size >= nextRollover) {
            rollOver();
        }
    }
   }

```



### 实现思路

> 现在知道了两个类是如何实现切分文件的了
> 然后在DailyRollingFileAppender类的基础上扩展，参考RollingFileAppender增加文件大小的判断。
> 将判断文件大小的逻辑加在subAppend方法中，当超过配置的size最大值时调用sizeRollOver()方法。

### 最终代码合并如下

```java
package com.iguozhe.wonhot.common.system;

import org.apache.log4j.DailyRollingFileAppender;
import org.apache.log4j.FileAppender;
import org.apache.log4j.Layout;
import org.apache.log4j.helpers.CountingQuietWriter;
import org.apache.log4j.helpers.LogLog;
import org.apache.log4j.helpers.OptionConverter;
import org.apache.log4j.spi.LoggingEvent;

import java.io.File;
import java.io.IOException;
import java.io.InterruptedIOException;
import java.io.Writer;
import java.text.SimpleDateFormat;
import java.util.*;

/**
 * 在DailyRollingFileAppender的基础上扩展
 * 支持同时按日期和大小切分log文件
 */
public class DailySizeRollingFileAppender extends FileAppender {

    // The code assumes that the following constants are in a increasing
    // sequence.
    static final int TOP_OF_TROUBLE=-1;
    static final int TOP_OF_MINUTE = 0;
    static final int TOP_OF_HOUR   = 1;
    static final int HALF_DAY      = 2;
    static final int TOP_OF_DAY    = 3;
    static final int TOP_OF_WEEK   = 4;
    static final int TOP_OF_MONTH  = 5;


    /**
     The date pattern. By default, the pattern is set to
     "'.'yyyy-MM-dd" meaning daily rollover.
     */
    private String datePattern = "'.'yyyy-MM-dd";

    /**
     The log file will be renamed to the value of the
     scheduledFilename variable when the next interval is entered. For
     example, if the rollover period is one hour, the log file will be
     renamed to the value of "scheduledFilename" at the beginning of
     the next hour.

     The precise time when a rollover occurs depends on logging
     activity.
     */
    private String scheduledFilename;

    /**
     The next time we estimate a rollover should occur. */
    private long nextCheck = System.currentTimeMillis () - 1;

    Date now = new Date();

    SimpleDateFormat sdf;

    RollingCalendar rc = new RollingCalendar();

    int checkPeriod = TOP_OF_TROUBLE;

    // The gmtTimeZone is used only in computeCheckPeriod() method.
    static final TimeZone gmtTimeZone = TimeZone.getTimeZone("GMT");


    /*****************start *********************/
    /**
     The default maximum file size is 10MB.
     */
    protected long maxFileSize = 10*1024*1024;

    /**
     There is one backup file by default.
     */
    protected int  maxBackupIndex  = 1;

    private long nextRollover = 0;
    /****************end **********************/

    /**
     The default constructor does nothing. */
    public DailySizeRollingFileAppender() {
    }

    /**
     Instantiate a <code>DailyRollingFileAppender</code> and open the
     file designated by <code>filename</code>. The opened filename will
     become the ouput destination for this appender.

     */
    public DailySizeRollingFileAppender (Layout layout, String filename,
                                         String datePattern) throws IOException {
        super(layout, filename, true);
        this.datePattern = datePattern;
        activateOptions();
    }

    /**
     The <b>DatePattern</b> takes a string in the same format as
     expected by {@link SimpleDateFormat}. This options determines the
     rollover schedule.
     */
    public void setDatePattern(String pattern) {
        datePattern = pattern;
    }

    /** Returns the value of the <b>DatePattern</b> option. */
    public String getDatePattern() {
        return datePattern;
    }


    /*******************************************/
    public void setMaxBackupIndex(int maxBackups) {
        this.maxBackupIndex = maxBackups;
    }
    public void setMaximumFileSize(long maxFileSize) {
        this.maxFileSize = maxFileSize;
    }
    public void setMaxFileSize(String value) {
        maxFileSize = OptionConverter.toFileSize(value, maxFileSize + 1);
    }
    protected void setQWForFiles(Writer writer) {
        this.qw = new CountingQuietWriter(writer, errorHandler);
    }
    public int getMaxBackupIndex() { return maxBackupIndex;}
    public long getMaximumFileSize() { return maxFileSize; }
    /*******************************************/

    public void activateOptions() {
        super.activateOptions();
        if(datePattern != null && fileName != null) {
            now.setTime(System.currentTimeMillis());
            sdf = new SimpleDateFormat(datePattern);
            int type = computeCheckPeriod();
            printPeriodicity(type);
            rc.setType(type);
            File file = new File(fileName);
            scheduledFilename = fileName+sdf.format(new Date(file.lastModified()));

        } else {
            LogLog.error("Either File or DatePattern options are not set for appender ["
                    +name+"].");
        }
    }

    void printPeriodicity(int type) {
        switch(type) {
            case TOP_OF_MINUTE:
                LogLog.debug("Appender ["+name+"] to be rolled every minute.");
                break;
            case TOP_OF_HOUR:
                LogLog.debug("Appender ["+name
                        +"] to be rolled on top of every hour.");
                break;
            case HALF_DAY:
                LogLog.debug("Appender ["+name
                        +"] to be rolled at midday and midnight.");
                break;
            case TOP_OF_DAY:
                LogLog.debug("Appender ["+name
                        +"] to be rolled at midnight.");
                break;
            case TOP_OF_WEEK:
                LogLog.debug("Appender ["+name
                        +"] to be rolled at start of week.");
                break;
            case TOP_OF_MONTH:
                LogLog.debug("Appender ["+name
                        +"] to be rolled at start of every month.");
                break;
            default:
                LogLog.warn("Unknown periodicity for appender ["+name+"].");
        }
    }


    // This method computes the roll over period by looping over the
    // periods, starting with the shortest, and stopping when the r0 is
    // different from from r1, where r0 is the epoch formatted according
    // the datePattern (supplied by the user) and r1 is the
    // epoch+nextMillis(i) formatted according to datePattern. All date
    // formatting is done in GMT and not local format because the test
    // logic is based on comparisons relative to 1970-01-01 00:00:00
    // GMT (the epoch).

    int computeCheckPeriod() {
        RollingCalendar rollingCalendar = new RollingCalendar(gmtTimeZone, Locale.getDefault());
        // set sate to 1970-01-01 00:00:00 GMT
        Date epoch = new Date(0);
        if(datePattern != null) {
            for(int i = TOP_OF_MINUTE; i <= TOP_OF_MONTH; i++) {
                SimpleDateFormat simpleDateFormat = new SimpleDateFormat(datePattern);
                simpleDateFormat.setTimeZone(gmtTimeZone); // do all date formatting in GMT
                String r0 = simpleDateFormat.format(epoch);
                rollingCalendar.setType(i);
                Date next = new Date(rollingCalendar.getNextCheckMillis(epoch));
                String r1 =  simpleDateFormat.format(next);
                //System.out.println("Type = "+i+", r0 = "+r0+", r1 = "+r1);
                if(r0 != null && r1 != null && !r0.equals(r1)) {
                    return i;
                }
            }
        }
        return TOP_OF_TROUBLE; // Deliberately head for trouble...
    }

    /**
     Rollover the current file to a new file.
     按时间切分
     */
    void rollOver() throws IOException {

        /* Compute filename, but only if datePattern is specified */
        if (datePattern == null) {
            errorHandler.error("Missing DatePattern option in rollOver().");
            return;
        }

        String datedFilename = fileName+sdf.format(now);
        // It is too early to roll over because we are still within the
        // bounds of the current interval. Rollover will occur once the
        // next interval is reached.
        if (scheduledFilename.equals(datedFilename)) {
            return;
        }

        // close current file, and rename it to datedFilename
        this.closeFile();

        File target  = new File(scheduledFilename);
        if (target.exists()) {
            target.delete();
        }

        File file = new File(fileName);
        boolean result = file.renameTo(target);//将log按指定格式重命名
        if(result) {
            LogLog.debug(fileName +" -> "+ scheduledFilename);
        } else {
            LogLog.error("Failed to rename ["+fileName+"] to ["+scheduledFilename+"].");
        }

        try {
            // This will also close the file. This is OK since multiple
            // close operations are safe.
            this.setFile(fileName, true, this.bufferedIO, this.bufferSize);
        }
        catch(IOException e) {
            errorHandler.error("setFile("+fileName+", true) call failed.");
        }
        scheduledFilename = datedFilename;
    }

    /**
     * 按文件大小切分
     */
    public void sizeRollOver(){
        File target;
        File file;
//        long size = ((CountingQuietWriter) qw).getCount();//size 没什么用了
//        LogLog.debug("rolling over count=" + size);
        //   if operation fails, do not roll again until
        //      maxFileSize more bytes are written
//        nextRollover = size + maxFileSize;  //在这这个字段没什么用了

        LogLog.debug("maxBackupIndex="+maxBackupIndex);

        String datedFilename = fileName + sdf.format(now);//用日期格式名 替换fileName

        boolean renameSucceeded = true;
        // If maxBackups <= 0, then there is no file renaming to be done.
        if(maxBackupIndex > 0) {//maxBackupIndex 大于0 时才可能需要重新命名
            // Delete the oldest file, to keep Windows happy.
            file = new File(/*fileName*/ datedFilename+ '.' + maxBackupIndex);
            if (file.exists())
                renameSucceeded = file.delete();

            // Map {(maxBackupIndex - 1), ..., 2, 1} to {maxBackupIndex, ..., 3, 2}
            for (int i = maxBackupIndex - 1; i >= 1 && renameSucceeded; i--) {//按照配置的最大编号递减 修改找到的当前文件编号+1  最终编号为1的文件修改为2空出编号1
                file = new File(/*fileName*/datedFilename + "." + i);
                if (file.exists()) {
                    target = new File(/*fileName*/datedFilename + '.' + (i + 1));
                    LogLog.debug("Renaming file " + file + " to " + target);
                    renameSucceeded = file.renameTo(target);
                }
            }

            if(renameSucceeded) {
                // Rename fileName to datedFilename.1
                target = new File(/*fileName*/datedFilename + "." + 1);

                this.closeFile(); // keep windows happy.

                file = new File(fileName);
                LogLog.debug("Renaming file " + file + " to " + target);
                renameSucceeded = file.renameTo(target);//fileName 改为finleName.1
                //
                //   if file rename failed, reopen file with append = true
                //
                if (!renameSucceeded) {
                    try {
                        this.setFile(fileName, true, bufferedIO, bufferSize);
                    }
                    catch(IOException e) {
                        if (e instanceof InterruptedIOException) {
                            Thread.currentThread().interrupt();
                        }
                        LogLog.error("setFile("+fileName+", true) call failed.", e);
                    }
                }
            }
        }

        //
        //   if all renames were successful, then
        //
        if (renameSucceeded) {
            try {
                // This will also close the file. This is OK since multiple
                // close operations are safe.
                this.setFile(fileName, false, bufferedIO, bufferSize);
//                nextRollover = 0;
            }
            catch(IOException e) {
                if (e instanceof InterruptedIOException) {
                    Thread.currentThread().interrupt();
                }
                LogLog.error("setFile("+fileName+", false) call failed.", e);
            }
        }
        scheduledFilename = datedFilename;
        ////
    }
    /**
     * This method differentiates DailyRollingFileAppender from its
     * super class.
     *
     * <p>Before actually logging, this method will check whether it is
     * time to do a rollover. If it is, it will schedule the next
     * rollover time and then rollover.
     * */
    protected void subAppend(LoggingEvent event) {
        long n = System.currentTimeMillis();
        if (n >= nextCheck) {
            now.setTime(n);
            nextCheck = rc.getNextCheckMillis(now);
            try {
                rollOver();
            }
            catch(IOException ioe) {
                if (ioe instanceof InterruptedIOException) {
                    Thread.currentThread().interrupt();
                }
                LogLog.error("rollOver() failed.", ioe);
            }
        }else if(fileName != null && qw != null){//添加文件大小的判断
            long size = ((CountingQuietWriter) qw).getCount();
            if (size >= maxFileSize && size >= nextRollover) {
                sizeRollOver();
            }
        }
        super.subAppend(event);
    }
}

/**
 *  RollingCalendar is a helper class to DailyRollingFileAppender.
 *  Given a periodicity type and the current time, it computes the
 *  start of the next interval.
 * */
class RollingCalendar extends GregorianCalendar {
    private static final long serialVersionUID = -3560331770601814177L;

    int type = DailySizeRollingFileAppender.TOP_OF_TROUBLE;

    RollingCalendar() {
        super();
    }

    RollingCalendar(TimeZone tz, Locale locale) {
        super(tz, locale);
    }

    void setType(int type) {
        this.type = type;
    }

    public long getNextCheckMillis(Date now) {
        return getNextCheckDate(now).getTime();
    }

    public Date getNextCheckDate(Date now) {
        this.setTime(now);

        switch(type) {
            case DailySizeRollingFileAppender.TOP_OF_MINUTE:
                this.set(Calendar.SECOND, 0);
                this.set(Calendar.MILLISECOND, 0);
                this.add(Calendar.MINUTE, 1);
                break;
            case DailySizeRollingFileAppender.TOP_OF_HOUR:
                this.set(Calendar.MINUTE, 0);
                this.set(Calendar.SECOND, 0);
                this.set(Calendar.MILLISECOND, 0);
                this.add(Calendar.HOUR_OF_DAY, 1);
                break;
            case DailySizeRollingFileAppender.HALF_DAY:
                this.set(Calendar.MINUTE, 0);
                this.set(Calendar.SECOND, 0);
                this.set(Calendar.MILLISECOND, 0);
                int hour = get(Calendar.HOUR_OF_DAY);
                if(hour < 12) {
                    this.set(Calendar.HOUR_OF_DAY, 12);
                } else {
                    this.set(Calendar.HOUR_OF_DAY, 0);
                    this.add(Calendar.DAY_OF_MONTH, 1);
                }
                break;
            case DailySizeRollingFileAppender.TOP_OF_DAY:
                this.set(Calendar.HOUR_OF_DAY, 0);
                this.set(Calendar.MINUTE, 0);
                this.set(Calendar.SECOND, 0);
                this.set(Calendar.MILLISECOND, 0);
                this.add(Calendar.DATE, 1);
                break;
            case DailySizeRollingFileAppender.TOP_OF_WEEK:
                this.set(Calendar.DAY_OF_WEEK, getFirstDayOfWeek());
                this.set(Calendar.HOUR_OF_DAY, 0);
                this.set(Calendar.MINUTE, 0);
                this.set(Calendar.SECOND, 0);
                this.set(Calendar.MILLISECOND, 0);
                this.add(Calendar.WEEK_OF_YEAR, 1);
                break;
            case DailySizeRollingFileAppender.TOP_OF_MONTH:
                this.set(Calendar.DATE, 1);
                this.set(Calendar.HOUR_OF_DAY, 0);
                this.set(Calendar.MINUTE, 0);
                this.set(Calendar.SECOND, 0);
                this.set(Calendar.MILLISECOND, 0);
                this.add(Calendar.MONTH, 1);
                break;
            default:
                throw new IllegalStateException("Unknown periodicity type.");
        }
        return getTime();
    }
}

```

### 配置文件

```properties

#dao-log logger的名字 info 日志级别 wonhot-dao-log为appender的名字可以多个，表示logger应用多个appender
log4j.logger.dao-log=info,wonhot-dao-log
#同时按日期和文件大小切分
log4j.appender.wonhot-dao-log = com.iguozhe.wonhot.common.system.DailySizeRollingFileAppender
log4j.appender.wonhot-dao-log.File = ../logs/wonhot-dao/log.log
log4j.appender.wonhot-dao-log.Append = true
log4j.appender.wonhot-dao-log.Threshold = info
log4j.appender.wonhot-dao-log.DatePattern = '.'yyyy-MM-dd-HH-mm 
log4j.appender.wonhot-dao-log.MaxFileSize = 1KB
log4j.appender.wonhot-dao-log.MaxBackupIndex = 100
log4j.appender.wonhot-dao-log.layout = org.apache.log4j.PatternLayout
log4j.appender.wonhot-dao-log.layout.ConversionPattern= %d{ISO8601} [%F:%L][%p]:%m%n

```







