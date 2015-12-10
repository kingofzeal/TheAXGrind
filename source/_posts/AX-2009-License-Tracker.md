title: AX 2009 License Tracker
date: 2015-06-15 07:00:00
tags:
 - X++
 - Security
 - Development
categories:
 - AX 2009
 - Reporting
---
As many of you probably know, AX 2009 uses a concurrent licensing model. Compared to 2012's access-based named-user model, 2009 only has a single pool of licenses, which are given to users on a first-come, first-serve basis. From my experience, the server will generally let you connect a few extra clients before users start receiving an error and the server denies the connection. If your company has more concurrent users than licenses available, there are only a few ways to help manage this situation:

1. Purchase additional licenses.
1. Set default timeouts on every user. When a client is inactive for the time specified by the timeout, the client is disconnected from the server (with a message to the user) and the client closed. 
1. Keep an admin user (or similar) logged in. When the license pool starts getting depleted, manually kick users offline. We have done this in the past, and we normally tried to target only multiple or old sessions that were not properly terminated. You can achieve the same result by restarting server processes sequentially, allowing users to reconnect to a different server when one is taken offline. Once all the servers are restarted, users that are not active or improperly disconnected will naturally no longer be consuming licenses.
 

The last option is obviously less desirable, but we have been in a position where it was the only route available to us. Like many organizations, there is a fairly detailed process to request capital expenditures. In this context, one of our biggest struggles was justifying the need for additional licenses. The same users were not always complaining about not being able to use the system, but as administrators we were hearing about it at least once a day, if not 2-3 times. However, we were not able to quantitatively provide evidence that we were running out of licenses, or how many we actually needed.

While we no longer have to deal with our license pool on a daily basis, we still keep the consequences of releasing new functionality or otherwise expanding the user base in the back of our mind, and we are careful to make sure that such expansion won't adversely affect our current users. To help with these kinds of decisions, I created a utility that examined our historical license usage over time.

The tool is based on İsmail Özcan's post [here](http://ismailozcan68.blogspot.com/2012/09/how-to-find-maximum-number-of-logged-on.html), with modifications to make it more versatile.

```axapta LicenseStats
class LicenseStats extends RunBaseBatch
{
    date            beginDate;
    date            endDate;
    timeofday       resolution;

    DialogField     fieldBeginDate,
                    fieldEndDate,
                    fieldResolution;

    #Define.CurrentVersion(1)

    #LocalMacro.CurrentList
    beginDate,
    endDate,
    resolution
    #EndMacro
}

public ClassDescription caption()
{
    return "Active Users Over Time";
}

protected Object dialog(DialogRunbase dialog, boolean forceOnClient)
{
    DialogRunbase   ret = super(dialog, forceOnClient);
    ;

    fieldBeginDate  = ret.addFieldValue(typeId(TransDate), prevMth(today()), "Begin Date");
    fieldEndDate    = ret.addFieldValue(typeId(TransDate), today(), "End Date");
    fieldResolution = ret.addFieldValue(typeId(TimeHour24), str2time("00:15"), "Resolution");

    return ret;
}

public boolean getFromDialog()
{
    ;

    beginDate   = fieldBeginDate.value();
    endDate     = fieldEndDate.value();
    resolution  = fieldResolution.value();

    return true;
}

public LicenseStatsTable getStats(boolean showProgress = true)
{
    #AviFiles

    SysuserLog              sysUserLog;
    UtcDateTime             dateTime,
                            loopDate;
    int                     i,
                            totalDays;
    int64                   n,
                            totalSeconds;
    LicenseStatsTable   res;
    ;

    dateTime = DateTimeUtil::newDateTime(beginDate, 0, DateTimeUtil::getUserPreferredTimeZone());
    totalDays = endDate - beginDate + 1;
    totalSeconds = totalDays * 86400;

    if (showProgress)
    {
        this.progressInit("Calculating", real2int((totalDays * 86400) / resolution) , #AviUpdate);
        progress.setCount(0);
    }

    for (n = 0; n < totalSeconds; n += resolution)
    {
        loopDate = DateTimeUtil::addSeconds(dateTime, n);

        if (showProgress)
            progress.setText(
                DateTimeUtil::toStr(
                    DateTimeUtil::applyTimeZoneOffset(
                        loopDate,
                        DateTimeUtil::getUserPreferredTimeZone()
                    )
                )
            );

        select count(RecId) from sysUserLog
            where sysUserLog.LogoutDateTime &&
                  sysUserLog.LogoutDateTime >= loopDate &&
                  sysUserLog.CreateDateTime <= loopDate;

        res.DateTime = loopDate;
        res.UserCount = int642int(sysUserLog.RecId);
        res.insert();

        if (showProgress) progress.incCount();
    }

    return res;
}

public container pack()
{
    return [#CurrentVersion, #CurrentList];
}

public date parmBeginDate(date _beginDate = beginDate)
{
    ;
    beginDate = _beginDate;

    return beginDate;
}

public date parmEndDate(date _endDate = endDate)
{
    ;
    endDate = _endDate;

    return endDate;
}

public TimeOfDay parmResolution(TimeOfDay _resolution = resolution)
{
    ;
    resolution = _resolution;

    return resolution;
}

public boolean unpack(container packedClass)
{
    int     version = RunBase::getVersion(packedClass);
    ;

    switch (version)
    {
        case #CurrentVersion:
            [version, #CurrentList] = packedClass;
            break;

        default:
            return false;
    }

    return true;
}

public static void main(Args _args)
{
    LicenseStatsTable   infoT;
    LicenseStats        stats = new LicenseStats();
    ;

    if (stats.prompt())
    {
        infoT = stats.getStats();
    }

    while select infoT
    {
        info(strfmt("%1: %2", infoT.DateTime, infoT.UserCount));
    }
}
 ```

There is also an associated temporary table go with this class, which has two fields: a UtcDateTime (DateTime) and Int (UserCount). An instance of this table is populated during the execution.

The primary function, `getStats`, is very similar to the original code. Instead of hard-coding dates and working backwards from the current day, I use a dialog to collect the start and end days and work forward from the beginning in a single loop. Additionally, I allow the 1-hour check time to be any amount of time, measured in seconds (up to 24 hours). I also use the current user's default Timezone Offset to determine the start and end times of the days, though I have noticed that Daylight Savings Time does tend to give one or two extra results if it lands in the middle of the requested range. I also fixed a bug that allowed users to be excluded from the count if they logged on or off at exactly one of the intervals.

I also added some additional framework to allow other objects to use the same processing, but there is no error checking or validation on inputs. Because the InfoLog can get rather extensive and difficult to handle, we've opted to consume this in a simple form that prompts the user for the start and end dates and resolution, and populates a grid with the result. From there, the user can export to Excel for further analysis or graphing.

The end results of this tool should be read as "At exactly time X, there were Y users logged in", and won't include any information about users who logged in and out between consecutive intervals. Generally speaking, a smaller resolution (eg, 1 second) will give you the greatest clarity, but since there are (60 * 60 * 24 =) 86400 seconds in a day, it will take a rather long time to process.