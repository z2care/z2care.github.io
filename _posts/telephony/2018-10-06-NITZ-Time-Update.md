---
layout: post
title: "NITZ时间同步流程"
date: 2018-10-06 23:56:00 +0800
categories: 分析与探索
tag: Telephony
---
* content
{:toc}

NITZ time update similar from ntp, but it have time zone, so we start to
 learn it.

<!-- more -->

# NITZ时间同步流程

```
sequenceDiagram
sprd_ril->>sprd_libril: RIL_onUnsolicitedResponse
sprd_libril->>sprd_libril: sendResponse
sprd_libril->>RILJ: RIL_UNSOL_NITZ_TIME_RECEIVED
RILJ->>RILJ: setOnNITZTime
RILJ->>GsmServiceStateTracker: EVENT_NITZ_TIME
GsmServiceStateTracker->>GsmServiceStateTracker: setTimeFromNITZString
```

## RILC层

### 1. 接收到AT上报
>++vendor/sprd/proprietories-source/ril/sprd_ril/sprd_ril.c +14034++

```
static void onUnsolicited (const char *s, const char *sms_pdu)
{
    ...
14034    if (strStartsWith(s, "+CTZV:")) {
14035        /*NITZ time */
14036        char *response;
14037        char *tmp;
14038        char *raw_str;
14039
14040        line = strdup(s);
14041        tmp = line;
14042        at_tok_start(&tmp);//获取：后的内容
14043
14044        err = at_tok_nextstr(&tmp, &response);//检查是否有多余内容
14045        if (err != 0) {
14046            RILLOGE("invalid NITZ line %s\n", s);
14047        } else {
14048#if defined (GLOBALCONFIG_RIL_SAMSUNG_LIBRIL_INTF_EXTENSION)
14049            raw_str = (char *)malloc(strlen(response) + 1);
14050            memcpy(raw_str, response, strlen(response) + 1)
14051            RIL_requestTimedCallback (onNitzReceived, raw_str, NULL);
14052#elif defined (RIL_SPRD_EXTENSION)
14053            **RIL_onUnsolicitedResponse** (//此函数是指针注册在RILD中
14054                    RIL_UNSOL_NITZ_TIME_RECEIVED,//上报给RILJ的ID
14055                    response, strlen(response)+1);//response是内容，长度加1是因为不含换行
14056#endif
14057        }
```

### 2. RIL_onUnsolicitedResponse处理
>++vendor/sprd/proprietories-source/ril/sprd_libril/sprd_ril.c +6322++

```
6326    int unsolResponseIndex = -1;
6327    int ret;
6328    int64_t timeReceived = 0;
...
6431    // Mark the time this was received, doing this
6432    // after grabing the wakelock incase getting
6433    // the elapsedRealTime might cause us to goto
6434    // sleep.
6435    if (unsolResponse == RIL_UNSOL_NITZ_TIME_RECEIVED) {
6436        timeReceived = elapsedRealtime();//记录此刻系统运行时间，用于校准NITZ时间
6437    }
...
6441    Parcel p;//打包数据给RILJ
6442
6443    p.writeInt32 (RESPONSE_UNSOLICITED);//data 0,上报类型是非主动请求
6444    p.writeInt32 (unsolResponse)//data 1,上报的信息ID
6445
6446    ret = s_unsolResponses[unsolResponseIndex]
6447                .responseFunction(p, data, datalen);//三个技巧用法
...
6468            p.writeInt64(timeReceived)//data 2,系统运行时间
...
6472    ret = **sendResponse(p, socket_type);**
6473    if (ret != 0 && unsolResponse == RIL_UNSOL_NITZ_TIME_RECEIVED) {//这里用于备份NITZ时间
6474
6475        // Unfortunately, NITZ time is not poll/update like everything
6476        // else in the system. So, if the upstream client isn't connected,
6477        // keep a copy of the last NITZ response (with receive time noted
6478        // above) around so we can deliver it when it is connected
...
6484
6485        s_lastNITZTimeData = malloc(p.dataSize());
6486        s_lastNITZTimeDataSize = p.dataSize();
6487        memcpy(s_lastNITZTimeData, p.data(), p.dataSize());
6488    }
...
```

### 3. 继续传递
> ++vendor/sprd/proprietories-source/ril/sprd_libril/sprd_ril.cpp++

```
2734static int
2735sendResponse (Parcel &p, RIL_SOCKET_TYPE socket_type) {
2736    printResponse;//打印log信息
2737    return sendResponseRaw(p.data(), p.dataSize(), socket_type);//传递打包后的信息
2738}
```

4. 写入节点
> ++vendor/sprd/proprietories-source/ril/sprd_libril/sprd_ril.cpp++
```
2687static int
2688sendResponseRaw (const void *data, size_t dataSize, RIL_SOCKET_TYPE socket_type) {
2689    int fd = s_rilSocketParam.fdCommand;//socket节
2690    int ret;
...
2709    pthread_mutex_lock(writeMutex);//加锁
2710
2711    header = htonl(dataSize);//序列话后顺序
2712
2713    ret = blockingWrite(fd, (void *)&header, sizeof(header));//写入size
...
2721    ret = blockingWrite(fd, data, dataSize)//写入data
2728
2729    pthread_mutex_unlock(writeMutex);//解锁
2730
2731    return 0;
2732}
```

## RILJ层

### 1. 处理上报消息
> ++frameworks/opt/telephony/src/java/com/android/internal/telephony/RIL.java++

```
2632    protected void
2633    processUnsolicited (Parcel p) {
...
2637        response = p.readInt();//读出消息ID
2638
...
2653            case RIL_UNSOL_NITZ_TIME_RECEIVED: ret =  responseString(p); break;//读出消息字串
...
2797                long nitzReceiveTime = p.readLong()//读出携带的系统运行时间
2798
2799                Object[] result = new Object[2];
2800
2801                result[0] = ret;
2802                result[1] = Long.valueOf(nitzReceiveTime);
2803
...
2807                if (ignoreNitz) {
2808                    if (RILJ_LOGD) riljLog("ignoring UNSOL_NITZ_TIME_RECEIVED");
2809                } else {
2810                    if (mNITZTimeRegistrant != null) {
2811
2812                        **mNITZTimeRegistrant**
2813                            .notifyRegistrant(new AsyncResult (null, result, null);//注意mNITZTimeRegistrant的初始化
2814                    } else {
2815                        // in case NITZ time registrant isnt registered yet
2816                        mLastNITZTimeInfo = result;
2817                    }
2818                }
2819            break;
```

### 2. 观察监听者是如何处理的
> ++frameworks/opt/telephony/src/java/com/android/internal/telephony/RIL.java++

```
665    @Override public void
666    setOnNITZTime(Handler h, int what, Object obj) {
667        **super.setOnNITZTime(h, what, obj);**//在父类BaseCommands.java里实例化
668
669        // Send the last NITZ time if we have it
670        if (mLastNITZTimeInfo != null) {
671            mNITZTimeRegistrant
672                .notifyRegistrant(
673                    new AsyncResult (null, mLastNITZTimeInfo, null));
674            mLastNITZTimeInfo = null;
675        }
676    }
```

### 3. 注册监听的GsmServiceStateTracker
> ++frameworks/opt/telephony/src/java/com/android/internal/telephony/gsm/GsmServiceStateTracker.java++

```
252    public GsmServiceStateTracker(GSMPhone phone) {
...
268        mCi.setOnNITZTime(this, EVENT_NITZ_TIME, null);//注册消息事件
...
333    @Override
334    public void handleMessage (Message msg) {
...
450            case EVENT_NITZ_TIME:
451                ar = (AsyncResult) msg.obj;
452
453                String nitzString = (String)((Object[])ar.result)[0];//读出携带的res
454                long nitzReceiveTime = ((Long)((Object[])ar.result)[1]).longValue();//读出携带的系统运行时间
455
456                setTimeFromNITZString(nitzString, nitzReceiveTime);
457                break;
```

### 4. 关键的设置时间函
> ++

```
1747    /**
1748     * nitzReceiveTime is time_t that the NITZ time was posted
1749     */
1750    private void setTimeFromNITZString (String nitz, long nitzReceiveTime) {
1751        // "yy/mm/dd,hh:mm:ss(+/-)tz"
1752        // tz is in number of quarter-hours
1753
1754        long start = SystemClock.elapsedRealtime();//记录当前系统运行时间
1755        if (DBG) {log("NITZ: " + nitz + "," + nitzReceiveTime +
1756                        " start=" + start + " delay=" + (start - nitzReceiveTime));
1757        }
1758
1759        try {
1760            /* NITZ time (hour:min:sec) will be in UTC but it supplies the timezone
1761             * offset as well (which we won't worry about until later) */
1762            Calendar c = Calendar.getInstance(TimeZone.getTimeZone("GMT"));
1763
1764            c.clear();
1765            c.set(Calendar.DST_OFFSET, 0);//夏令时偏移0
1766
1767            String[] nitzSubs = nitz.split("[/:,+-]");
1768
1769            try {
1770                int year = 2000 + Integer.parseInt(nitzSubs[0]);
1771                c.set(Calendar.YEAR, year);
1772
1773                // month is 0 based!
1774                int month = Integer.parseInt(nitzSubs[1]) - 1;
1775                c.set(Calendar.MONTH, month);
1776
1777                int date = Integer.parseInt(nitzSubs[2]);
1778                c.set(Calendar.DATE, date);
1779
1780                int hour = Integer.parseInt(nitzSubs[3]);
1781                c.set(Calendar.HOUR, hour);
1782
1783                int minute = Integer.parseInt(nitzSubs[4]);
1784                c.set(Calendar.MINUTE, minute);
1785
1786                int second = Integer.parseInt(nitzSubs[5]);
1787                c.set(Calendar.SECOND, second);
1788            } catch (NumberFormatException e) {
1789                log("NITZ: time format is incorrect");
1790                // set nitzReceiveTime to an invalid value to bypass the time update
1791                nitzReceiveTime = SystemClock.elapsedRealtime() + 60 * 1000;
1792            }//时间分割完毕
1793
1794            boolean sign = (nitz.indexOf('-') == -1);//时区偏移方向
1795
1796            int tzOffset = Integer.parseInt(nitzSubs[6]);//时区偏移值
1797
1798            int dst = (nitzSubs.length >= 8 ) ? Integer.parseInt(nitzSubs[7])
1799                                              : 0;//夏令时偏移
1800            if (DBG) log("sign = " + sign + "; tzOffset = " + tzOffset + "; dst = " + dst);
1801
1802            // The zone offset received from NITZ is for current local time,
1803            // so DST correction is already applied.  Don't add it again.
1804            //
1805            // tzOffset += dst * 4;
1806            //
1807            // We could unapply it if we wanted the raw offset.
1808
1809            tzOffset = (sign ? 1 : -1) * tzOffset * 15 * 60 * 1000;//时区偏移
1810
1811            TimeZone    zone = null;
1812
1813            // As a special extension, the Android emulator appends the name of
1814            // the host computer's timezone to the nitz string. this is zoneinfo
1815            // timezone name of the form Area!Location or Area!Location!SubLocation
1816            // so we need to convert the ! into /
1817            if (nitzSubs.length >= 9) {//转换时区信息
1818                String  tzname = nitzSubs[8].replace('!','/');
1819                if (DBG) log("tzname = " + tzname);
1820                zone = TimeZone.getTimeZone( tzname );
1821            }
1822
1823            String operatorIsoCountryProperty = TelephonyManager.getProperty(
1824                    TelephonyProperties.PROPERTY_OPERATOR_ISO_COUNTRY, mPhone.getPhoneId());//以下是时区设置
1825            String iso = SystemProperties.get(operatorIsoCountryProperty);
1826
1827            if (DBG) log("iso = " + iso);
1828
1829            if (zone == null) {
1830
1831                if (mGotCountryCode) {
1832                    if (iso != null && iso.length() > 0) {
1833                        zone = TimeUtils.getTimeZone(tzOffset, dst != 0,
1834                                c.getTimeInMillis(),
1835                                iso);
1836                    } else {
1837                        // We don't have a valid iso country code.  This is
1838                        // most likely because we're on a test network that's
1839                        // using a bogus MCC (eg, "001"), so get a TimeZone
1840                        // based only on the NITZ parameters.
1841                        zone = getNitzTimeZone(tzOffset, (dst != 0), c.getTimeInMillis());
1842                    }
1843                }
1844            }
1845
1846            if ((zone == null) || (mZoneOffset != tzOffset) || (mZoneDst != (dst != 0))){
1847                // We got the time before the country or the zone has changed
1848                // so we don't know how to identify the DST rules yet.  Save
1849                // the information and hope to fix it up later.
1850
1851                mNeedFixZoneAfterNitz = true;
1852                mZoneOffset  = tzOffset;
1853                mZoneDst     = dst != 0;
1854                mZoneTime    = c.getTimeInMillis();
1855            }
1856
1857            if (zone != null) {
1858                if (getAutoTimeZone()) {
1859                    setAndBroadcastNetworkSetTimeZone(zone.getID());
1860                }
1861                saveNitzTimeZone(zone.getID());
1862            }
1863
1864            String ignore = SystemProperties.get("gsm.ignore-nitz");
1865            if (ignore != null && ignore.equals("yes")) {
1866                log("NITZ: Not setting clock because gsm.ignore-nitz is set");
1867                return;
1868            }
1869
1870            try {//处理时间设置
1871                mWakeLock.acquire();
1872
1873                if (getAutoTime()) {
1874                    long millisSinceNitzReceived
1875                            = SystemClock.elapsedRealtime() - nitzReceiveTime;
1876
1877                    if (millisSinceNitzReceived < 0) {
1878                        // Sanity check: something is wrong
1879                        if (DBG) {
1880                            log("NITZ: not setting time, clock has rolled "
1881                                            + "backwards since NITZ time was received, "
1882                                            + nitz);
1883                        }
1884                        return;
1885                    }
1886
1887                    if (millisSinceNitzReceived > Integer.MAX_VALUE) {
1888                        // If the time is this far off, something is wrong > 24 days!
1889                        if (DBG) {
1890                            log("NITZ: not setting time, processing has taken "
1891                                        + (millisSinceNitzReceived / (1000 * 60 * 60 * 24))
1892                                        + " days");
1893                        }
1894                        return;
1895                    }
1896
1897                    // Note: with range checks above, cast to int is safe
1898                    c.add(Calendar.MILLISECOND, (int)millisSinceNitzReceived);
1899
1900                    if (DBG) {
1901                        log("NITZ: Setting time of day to " + c.getTime()
1902                            + " NITZ receive delay(ms): " + millisSinceNitzReceived
1903                            + " gained(ms): "
1904                            + (c.getTimeInMillis() - System.currentTimeMillis())
1905                            + " from " + nitz);
1906                    }
1907
1908                    setAndBroadcastNetworkSetTime(c.getTimeInMillis());//此处设置时间
1909                    Rlog.i(LOG_TAG, "NITZ: after Setting time of day");
1910                }
1911                SystemProperties.set("gsm.nitz.time", String.valueOf(c.getTimeInMillis()));
1912                saveNitzTime(c.getTimeInMillis());
1913                if (VDBG) {
1914                    long end = SystemClock.elapsedRealtime();
1915                    log("NITZ: end=" + end + " dur=" + (end - start));
1916                }
1917                mNitzUpdatedTime = true;
1918            } finally {
1919                mWakeLock.release();
1920            }
1921        } catch (RuntimeException ex) {
1922            loge("NITZ: Parsing NITZ time " + nitz + " ex=" + ex);
1923        }
1924    }
...
1974    /**
1975     * Set the time and Send out a sticky broadcast so the system can determine
1976     * if the time was set by the carrier.
1977     *
1978     * @param time time set by network
1979     */
1980    private void setAndBroadcastNetworkSetTime(long time) {
1981        if (DBG) log("setAndBroadcastNetworkSetTime: time=" + time + "ms");
1982        SystemClock.setCurrentTimeMillis(time);//调用AlarmManagerService的setTime设置时间
1983        Intent intent = new Intent(TelephonyIntents.ACTION_NETWORK_SET_TIME);
1984        intent.addFlags(Intent.FLAG_RECEIVER_REPLACE_PENDING);
1985        intent.putExtra("time", time);
1986        mPhone.getContext().sendStickyBroadcastAsUser(intent, UserHandle.ALL);//用于NetworkTimeUpdateService.java记录本次设置的时间
1987    }
```