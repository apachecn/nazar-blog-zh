# Java 中不一致的日期处理

> 原文：<http://web.archive.org/web/20220810161336/https://imrannazar.com/Discordian-Date-Handling-in-Java>

日期的表示可以追溯到几千年前:每个传统和宗教都有自己的表示和计算日历的方式，很少有简单的方法在日历之间移动。在一个日历中计算一个日期，在另一个日历中给定相同的日期，可能涉及一系列基于月亮的相位、轨道倾角和其他类似事情的费力的操作。

### 不一致的历法

Discordianism 使用的日历灵感来自公认的公历，突出数字 5。与十二个月相反，一年有五个季节，每个季节有固定的天数:

| 季节 | 天 |
| --- | --- |
| 混乱 | Seventy-three |
| 不调和 | Seventy-three |
| 混乱 | Seventy-three |
| 官僚主义 | Seventy-three |
| 余波 | Seventy-three |

*Table 1: Seasons of the Discordian calendar*

这导致一年有 365 天，与公历一致；因此，非公历中的某一天总是与公历中的同一天相对应。

除了有五个季节之外，每个星期由五天组成:甜蜜的早晨，繁荣的时候，Pungenday，皮刺-皮刺和设置橙色。因为日历与公历对齐，每个不和谐的一年由 73 周 5 天组成；因此，日历中的每一天总是具有相同的日名和日期。

在公历中，闰日在 400 年中的 97 年中加入，以 4 年为一个周期。同样的过程也适用于不和谐音，圣提卜节被插在混乱的第 59 和 60 天(2 月 28 日和 3 月 1 日)之间。

最后一个细节是，公元前 1166 年开始的非基督教历法；从那时起，年份就与公历步调一致，并被标记为 *anno discordia* 或“不和谐圣母年”。下面是两种日历中日期的几个例子。

#### 公历和公历中的日期示例

```
Chaos 1st, 3000             = January 1st, 1834
Bureaucracy 70th, 3155      = October 16th, 1989
St. Tib's Day, 3178         = February 29th, 2012
```

### 在日历之间转换

因为不太和历非常有规律，所以不太和历日期之间的转换相对简单。所有需要做的就是计算年、日、月的偏移量。

#### 给定年份和日期的不一致日期计算

```
DYear = Year + 1166

; Handle leap years
IF Year is-a-leap-year THEN
    IF Day = 59 THEN
        DSeasonday = "St. Tib's Day"
    ELSE IF Day > 59
 ; Days after Feb 29th need to be shifted up to make this
	; year into a regular 365-day year, for calculation purposes
	Day = Day - 1
    END IF
END IF

SeasonNames = ["Chaos", "Discord", "Confusion", "Bureaucracy", "The Aftermath"]
DayNames = ["Sweetmorn", "Boomtime", "Pungenday", "Prickle-Prickle", "Setting Orange"]

IF DSeasonday is-not-already-set THEN
    DSeason = SeasonNames[Day / 73]
    DWeekday = DayNames[Day MOD 5]
    DSeasonday = Day MOD 73
END IF
```

上面是一个将日期转换成不一致的伪代码示例，并且考虑了闰年的特殊情况。从公历日期转换回公历日期同样简单。唯一复杂的是闰年的情况，在这种情况下，不一致历法所报告的日期会比其他年份提前一天。

#### 从公历计算公历日/年

```
Year = DYear - 1166
Day = (DSeasonNum - 1) * 73 + DSeasondayNum

IF Year is-a-leap-year THEN
    IF DSeasonday = "St. Tib's Day" THEN
        Day = 60
    ELSE IF Day >= 60
        Day = Day + 1
    END IF
END IF
```

### Java 实现

用 Java 编写上述算法变得非常简单，因为有了日期/日历计算类`java.util.Calendar`；特别是，`GregorianCalendar`子类允许以快速有效的方式计算闰年。下面的代码实现了从一个日历到另一个日历的转换，在两种情况下都提供了日期的可读表示。

#### ddate.java:公历/公历日期转换

```
import java.util.Date;
import java.util.Calendar;
import java.util.GregorianCalendar;

public class ddate
{
    private int _year, _season, _yearDay, _seasonDay, _weekDay;
    private boolean _isLeap;

    private String[] _seasonNames = {"Chaos","Discord","Confusion","Bureaucracy","The Aftermath"};
    private String[] _dayNames = {"Sweetmorn","Boomtime","Pungenday","Prickle-Prickle","Setting Orange"};

    public ddate(Date d)
    {
        GregorianCalendar gc = new GregorianCalendar();
	gc.setTime(d);

	_year = gc.get(Calendar.YEAR) + 1166;
	_yearDay = gc.get(Calendar.DAY_OF_YEAR);
	_isLeap = gc.isLeapYear(gc.get(Calendar.YEAR));

	int yd = _yearDay - 1;

	if(_isLeap && yd > 59)
	    yd--;

	_season = (yd / 73) + 1;
	_weekDay = (yd % 5) + 1;
	_seasonDay = (yd % 73) + 1;
    }

    public int getYear()          { return _year; }
    public int getSeason()        { return _season; }
    public int getYearDay()       { return _yearDay; }
    public int getSeasonDay()     { return _seasonDay; }
    public String getSeasonName() { return _seasonNames[_season-1]; }
    public String getDayName()    { return _dayNames[_yearDay-1]; }

    public String toString()
    {
        if(_isLeap && _yearDay == 59)
	{
	    return "St. Tib's Day, " + Integer.toString(_year);
	}
	else
	{
            return _dayNames[_weekDay-1] + ", " +
	           _seasonNames[_season-1] + " " +
	           Integer.toString(_seasonDay) + ", " +
	           Integer.toString(_year);
	}
    }

    public Date getTime()
    {
        GregorianCalendar gc = new GregorianCalendar();
	gc.set(Calendar.YEAR, _year - 1166);
	gc.set(Calendar.DAY_OF_YEAR, _yearDay);

	return gc.getTime();
    }
}
```

#### ddate.java 用法示例

```
Calendar foo = new Calendar();
foo.setTime(new Date());
foo.add(Calendar.DAY, -1);

ddate bar = new ddate(foo.getTime());
System.out.println("Yesterday was " + bar.toString());
```

### 扩展转换

上面详述的转化过程不包括非基督教日历的十个神圣的日子:一个神圣的日子落在每个季节的 5 号和 50 号。因为这些日子经常发生，所以为其提供一个接口并没有带来额外的困难，为了简洁起见，上面的代码中已经省略了这样一个接口。

tf@imrannazar.comT5，2010 年 3 月。

*文章日期:2010 年 3 月 8 日*