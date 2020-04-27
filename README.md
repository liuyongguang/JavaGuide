<!-- TOC depthFrom:1 depthTo:6 withLinks:1 updateOnSave:1 orderedList:0 -->

		- [排班逻辑梳理](#排班逻辑梳理)
			- [数据前置处理](#数据前置处理)
			- [刷状态逻辑](#刷状态逻辑)
					- [1、组装DefinedTimeLine](#1组装definedtimeline)
					- [2、组装DataKey key=companyId-employeeId-date](#2组装datakey-keycompanyid-employeeid-date)
					- [3、调用yank模块](#3调用yank模块)
					- [4、更新attendance_daily_status_detail表](#4更新attendancedailystatusdetail表)
					- [5、更新attendance_month_brief表](#5更新attendancemonthbrief表)
					- [6、发送TransactionMessageEvent事件](#6发送transactionmessageevent事件)
					- [7、发送假勤固化事件](#7发送假勤固化事件)
					- [8、数据改动事件](#8数据改动事件)

<!-- /TOC -->
### 排班逻辑梳理

-   入口：AttendanceTimingController#timingUpdateAttendanceRecordForScheduling

---

#### 数据前置处理

```java
public void updateAttendanceRecordForScheduling() {
        // currentTime -> 2020.04.27
        java.util.Date currentTime = new java.util.Date();
        // yesterdayDate -> 20202.03.27
        java.sql.Date yesterdayDate = DateUtil.getSqlDateByDate(DateUtil.getDateBefore(currentTime, 31));

        //获取此时间段所有未更新考勤状态的排班安排
        List<AttendanceSchedulingPO> allSchedulings = attendanceSchedulingDAO.selectByDateRange(yesterdayDate,
                DateUtil.getSqlDateByDate(currentTime), 0);

        // 这里每个考勤排班循环去拿班次设置，可以考虑改成批量一次拿所有的再组装
        iAttendanceSchedulingService.fillClockSetting(allSchedulings);
        if (allSchedulings == null || allSchedulings.size() == 0) return;

        //员工排班列表MAP<employeeId, 员工所有考勤排班>
        Map<String, List<AttendanceSchedulingPO>> employeeSchedulingMap = new HashedMap();
        // 这个判断了两次，上面已经判断了
        if (allSchedulings == null || allSchedulings.size() == 0) {
            return;
        }


        for (AttendanceSchedulingPO scheduling : allSchedulings) {
            AttendanceClockSettingPO clockSetting = scheduling.getClockSetting();
            if (clockSetting == null) continue;

            //判定开始时间 考勤日期+班次设置中上班时间(startingtime)往前推4个小时
            java.util.Date judgeStartTime = attendanceClockSettingService.getJudgeStartTime(clockSetting, scheduling
                    .getScheduleDate());

            //判定结束时间，考勤日期+班次设置中下班时间(closingtime)往后推6个小时
            java.util.Date judgeEndTime = attendanceClockSettingService.getJudgeEndTime(clockSetting, scheduling
                    .getScheduleDate());

            //判定结束时间在当前时间之前才有效
            if (judgeEndTime.compareTo(currentTime) <= 0) {
                List<AttendanceSchedulingPO> schedulings = employeeSchedulingMap.get(scheduling.getEmployeeId());
                if (schedulings == null) {
                    schedulings = new ArrayList<>();
                    employeeSchedulingMap.put(scheduling.getEmployeeId(), schedulings);
                }
                scheduling.setJudgeStartTime(judgeStartTime);
                scheduling.setJudgeEndTime(judgeEndTime);
                schedulings.add(scheduling);
            }
        }
        updateAttendanceRecordForSchedulingByParam(null, employeeSchedulingMap, yesterdayDate, currentTime);
    }
```

```java
private void updateAttendanceRecordForSchedulingByParam(String companyId, Map<String, List<AttendanceSchedulingPO>>
            employeeSchedulingMap, java.util.Date startDate, java.util.Date endDate, boolean refreshStat) {

        if (!employeeSchedulingMap.isEmpty()) {
            // 拿到所有员工的id
            ArrayList<String> employeeIds = new ArrayList<>(employeeSchedulingMap.keySet());

            // 获取这些员工这时段所有的考勤记录，每个员工每天一条记录
            List<AttendanceRecordPO> attendanceRecordPOs = attendanceRecordDAO
                    .selectByEmployeeIdsBetweenStartAndEndDate(employeeIds, DateUtil.getSqlDateByDate(startDate), DateUtil
                            .getSqlDateByDate(endDate));

            //考勤记录map，key为employeeId+datekey
            Map<String, AttendanceRecordPO> attendanceRecordPOMap = new HashedMap();
            for (AttendanceRecordPO attendanceRecordPO : attendanceRecordPOs) {
                attendanceRecordPOMap.put(attendanceRecordPO.getEmployeeId() + "-" + DateUtil.getDateInt
                        (attendanceRecordPO.getAttendanceDate()), attendanceRecordPO);
            }

            Set<Integer> planIdSet = new HashSet<>();
            List<AttendancePlanSettingPO> attendancePlanSettingPOs = new ArrayList<>();

            // 根据排班记录找出找出所有的考勤方案(attendancePlanSettingPOs)
            for (Map.Entry<String, List<AttendanceSchedulingPO>> entry : employeeSchedulingMap.entrySet()) {
                for (AttendanceSchedulingPO schedulingPO : entry.getValue()) {
                    if (!planIdSet.contains(schedulingPO.getPlanId())) {
                        planIdSet.add(schedulingPO.getPlanId());
                        AttendancePlanSettingPO settingPO = planSettingService.getSimplePlanSettingByPlanId
                                (schedulingPO.getPlanId());
                        if (settingPO != null) attendancePlanSettingPOs.add(settingPO);
                    }
                }
            }

            // 将所有考勤方案转成map: key=方案id
            Map<String, AttendancePlanSettingPO> planSettingPOMap = attendancePlanSettingService.convertToPOMap
                    (attendancePlanSettingPOs);

            List<DailyStatusDetailChangedInfo> changedInfos = new ArrayList<>();

            Map<String, EmployeeFixedDO> employeeFixedDOMap = new HashMap<>();
            Map<String, ClockPlanDetailDO> clockPlanMapByEmployeeIds = new HashMap<>();
            // 定时任务传的companyId=null ，此处不执行
            if (companyId != null) {
                employeeFixedDOMap = employeeDetailService.getEmployeeByIds(employeeIds, companyId);
                clockPlanMapByEmployeeIds = clockPlanService.getClockPlanMapByEmployeeIds(companyId, employeeIds);
            }

            //新增考勤记录，遍历每一个员工
            for (String employeeId : employeeIds) {
                List<AttendanceSchedulingPO> attendanceSchedulingPOs = employeeSchedulingMap.get(employeeId);
                if (attendanceSchedulingPOs == null) {
                    continue;
                }
                // 遍历每一个排班计划
                for (AttendanceSchedulingPO schedulingPO : attendanceSchedulingPOs) {
                    try {
                      // 拿到排班这一天对应的考勤记录
                        AttendanceRecordPO recordPO = attendanceRecordPOMap.get(schedulingPO.getEmployeeId() + "-" +
                                DateUtil.getDateInt(schedulingPO.getScheduleDate()));
                        //获取打卡设置，若考勤方案不存在，则打卡设置为空；若考勤方案中打卡按钮关闭，则为不打卡
                        AttendanceClockSettingPO clockSetting = schedulingPO.getClockSetting();

                        AttendancePlanSettingPO settingPO = planSettingPOMap.get(schedulingPO.getPlanId() + "");

                        // 获取这个排班日期内所有的请假详情记录
                        Map<Integer, List<AttendanceDailyStatusDetailPO>> dailymap =
                                iAttendanceDailyStatusDetailInCommonInCommon.getEmployeeDailyStatusDetailMap(employeeId,
                                        null, schedulingPO.getScheduleDate(), schedulingPO.getScheduleDate());

                        //找出 应工作时长 通过打卡设置
                        int shouldWorkMinute = AttendanceWorkTimeHelper.getWorkTimeOnDayBySetting(schedulingPO
                                .getClockSetting());
                        double shouldWorkHour = shouldWorkMinute * 1.0 / 60;

                        // 如果是排班按天考勤
                        if (settingPO.getPlanType() == EAttendancePlanType.SCHEDULING.getValue() && settingPO
                                .getPayAttendanceType() == EPayAttendanceType.ATTENDANCE_TYPE_DAY.getValue()) {
                            for (List<AttendanceDailyStatusDetailPO> attendanceDailyStatusDetailPOList : dailymap
                                    .values()) {
                                // 遍历这天的所有请假记录
                                for (AttendanceDailyStatusDetailPO attendanceDailyStatusDetailPO :
                                        attendanceDailyStatusDetailPOList) {
                                    // TODO 如果是假期&按天请假&请假时长是0
                                    if (attendanceDailyStatusDetailPO.getIsHoliday() == 1 &&
                                            attendanceDailyStatusDetailPO.getTimeUnit() == EAttendanceLeaveTimeUnit.DAY
                                                    .getValue() && attendanceDailyStatusDetailPO.getHours() == 0) {
                                        if (attendanceDailyStatusDetailPO.getStartTime() == 0 &&
                                                attendanceDailyStatusDetailPO.getEndTime() == 12) {
                                            // 请假上半天
                                            attendanceDailyStatusDetailPO.setSalaryDay(0.5);
                                            attendanceDailyStatusDetailPO.setHours(shouldWorkHour / 2);
                                        } else if (attendanceDailyStatusDetailPO.getStartTime() == 0 &&
                                                attendanceDailyStatusDetailPO.getEndTime() == 24) {
                                            // 请假全天
                                            attendanceDailyStatusDetailPO.setSalaryDay(1d);
                                            attendanceDailyStatusDetailPO.setHours(shouldWorkHour);
                                        } else if (attendanceDailyStatusDetailPO.getStartTime() == 12 &&
                                                attendanceDailyStatusDetailPO.getEndTime() == 24) {
                                            // 请假下半天
                                            attendanceDailyStatusDetailPO.setSalaryDay(0.5);
                                            attendanceDailyStatusDetailPO.setHours(shouldWorkHour / 2);
                                        }
                                    }
                                }
                            }
                        }

                        final UpdateSituationResult[] updateSituationResult = {null};
                        AttendanceRecordPO finalRecordPO;
                        AttendanceClockSituationDO clockSituationDO;

                        // 如果这天没有班次设置
                        if (clockSetting == null) {

                            // 没有考勤记录
                            if (recordPO == null) {
                              // 生成一个考勤记录，签到签退时间均为0
                                recordPO = dealAttendanceRecord(schedulingPO.getCompanyId(), employeeId, schedulingPO
                                        .getScheduleDate(), recordPO, null, 0, 0);
                            }
                            if (recordPO.getId() <= 0) {
                              // 如果是空的再去找一遍？？ recordPO已经是这个员工这天的考勤记录了，为什么这里还要在找一遍？
                                AttendanceRecordPO po = attendanceRecordDAO.selectByEmployeeIdAndDate(schedulingPO
                                        .getScheduleDate(), schedulingPO.getEmployeeId());
                                if (po != null) {
                                    recordPO = po;
                                }
                            }
                        } else {
                            recordPO = dealAttendanceRecord(schedulingPO.getCompanyId(), employeeId, schedulingPO
                                    .getScheduleDate(), recordPO, clockSetting, 0, 0);
                            if (recordPO.getId() <= 0) {
                                AttendanceRecordPO po = attendanceRecordDAO.selectByEmployeeIdAndDate(schedulingPO
                                        .getScheduleDate(), schedulingPO.getEmployeeId());
                                if (po != null) {
                                    recordPO = po;
                                }
                            }
                        }

                        clockSituationDO = iAttendanceClockRecordService
                                .getClockSituationByEmployeeAndDate(recordPO.getCompanyId(), employeeId, recordPO
                                        .getAttendanceDate(), null, null, settingPO);
                        if (null != clockSituationDO) {
                            recordPO.setSituationSnapshot(JsonUtil.toJson(clockSituationDO));
                        }
                        save(recordPO);
                        ClockPlanDetailDO clockPlanDetailDO = null;
                        EmployeeFixedDO employee = null;
                        // 排班定时任务特殊处理下
                        if (companyId == null) {
                            employee = iEmployeeDetailService.getEmployeeWithoutFilter(employeeId, EEmployeeStatus.STATUS_ON_AND_OFF.getValue(), schedulingPO.getCompanyId());
                            clockPlanDetailDO = clockPlanService.getClockPlanByEmployeeId(schedulingPO.getCompanyId(), employeeId);
                        } else {
                            clockPlanDetailDO = clockPlanMapByEmployeeIds.getOrDefault(employeeId, null);
                            employee = employeeFixedDOMap.getOrDefault(employeeId, null);
                        }

                        EmployeeBasicInfo employeeBasicInfo = employee != null ? new EmployeeBasicInfo(employee.getCompanyId(), employee.getEmployeeId(), null, employee.getEntryDate(), employee.getDismissionDate()) : null;
                        finalRecordPO = recordPO;
                        ClockPlanDetailDO finalClockPlanDetailDO = clockPlanDetailDO;

                        // 更新显示状态，管理员手工改动的不更新
                        transactionTemplate.execute(new TransactionCallback<Object>() {
                            @Override
                            public Object doInTransaction(TransactionStatus transactionStatus) {
                                updateSituationResult[0] = updateSituation(employeeBasicInfo, finalRecordPO,
                                        clockSituationDO, finalClockPlanDetailDO, dailymap.get(finalRecordPO.getId()), false, schedulingPO.getClockSettingId() > 0);
                                //排班方案置为已更新
                                schedulingPO.setIsUpdate(1);
                                attendanceSchedulingDAO.setUpate(schedulingPO.getPlanId(), schedulingPO.getEmployeeId(), schedulingPO.getScheduleDate(), 1);
                                return null;
                            }
                        });

                        if (updateSituationResult[0] != null) {
                            List<DailyStatusDetailChangedInfo> dailyStatusDetailChangedInfos =
                                    situationGeneratorForEngine.generateChangeDetail(updateSituationResult[0]);
                            changedInfos.addAll(dailyStatusDetailChangedInfos);
                        }
                    } catch (OperationFailedException e) {
                        LOGGER.info("排班刷状态业务异常, companyId : {} , exception : {}", companyId,e);
                    } catch (Exception e) {
                        LOGGER.error("排班刷状态未知异常, msg : {}, exception : {} ", e.getMessage(), e);
                    }
                }
                //排班刷完状态计算考勤数据
                try {
                    if (org.apache.commons.collections4.CollectionUtils.isNotEmpty(attendanceSchedulingPOs)) {
                        publisher.publishEvent(new AttendanceServiceDateSolidEvent(this, attendanceSchedulingPOs.get(0).getCompanyId(), Lists.newArrayList(employeeId)));
                    }
                } catch (Exception e) {
                    LOGGER.error("考勤服务假勤团队数据固化:失败统计:employeeId={}", employeeId);
                }
            }
            if (refreshStat) {
                if (changedInfos.size() > 0) {
                    Map<String, List<DailyStatusDetailChangedInfo>> collect = changedInfos.stream().collect
                            (Collectors.groupingBy(p -> p.getCompanyId()));

                    for (String _companyId : collect.keySet()) {
                        List<DailyStatusDetailChangedInfo> changedInfos1 = collect.get(_companyId);
                        SyncEngineMqMsgEvent syncEngineMqMsgEvent = situationGeneratorForEngine
                                .generateSyncEngineMqMsgEvent(_companyId, changedInfos1);
                        //保存排班时，原有逻辑一共调用了3次固化信息，现改为调用一次，只有整体处完排班后，调用固化逻辑，所以以下暂时注释掉
                        publisher.publishEvent(syncEngineMqMsgEvent);
                    }
                }
            }
        }
    }
```

---

#### 刷状态逻辑

最后返回AttendanceDailyStatusDetailRefreshModel

```java
public AttendanceDailyStatusDetailRefreshModel refreshFromEngine(EmployeeBasicInfo employeeBasicInfo, AttendanceRecordPO recordPO, AttendanceClockSituationDO clockSituationDO,
                                                                 ClockPlanDetailDO clockPlanDetailDO, boolean isworkDay) {
                                                                 }
```

###### 1、组装DefinedTimeLine

包括方案类型，每时段开始结束时间，最早最晚签到时间，豁免弹性时间，工作时长。

```java
  DefinedTimeLine definedTimeLine = new DefinedTimeLine();
        definedTimeLine.setCompanyId(companyId);
        definedTimeLine.setClockType(ClockType.None);
        definedTimeLine.setDate(new Date(attendanceDate.getTime()));

        int planTypeIntValue = clockSituationDO.getPlanSettingPO().getPlanType();
        PlanType planType = planTypeIntValue == EAttendancePlanType.SCHEDULING.getValue() ? PlanType.SCHEDULING : PlanType.FIXED;
        // 设置排班类型
        definedTimeLine.setPlanType(planType);

        int isClocking = 0;//
        AttendanceClockSettingPO clockSettingPO = clockSituationDO.getClockSettingPO();
        if (null != clockSettingPO
                && clockSettingPO.getTimeRanges() != null
                && clockSettingPO.getTimeRanges().size() > 0) {
            isClocking = clockSettingPO.getIsClocking();
            String startingtimeForFirst = clockSettingPO.getTimeRanges().get(0).getStartingtime();
            String closingtimeForFirst = clockSettingPO.getTimeRanges().get(0).getClosingtime();

            String startingtimeForSecond = closingtimeForFirst;
            String closingtimeForSecond = closingtimeForFirst;
            if (clockSettingPO.getTimeRanges().size() > 1) {
                startingtimeForSecond = clockSettingPO.getTimeRanges().get(1).getStartingtime();
                closingtimeForSecond = clockSettingPO.getTimeRanges().get(1).getClosingtime();
            }

            // 设置时段的开始时间结束时间，一天两卡First和Second相同
            definedTimeLine.setDateFromForFirst(startingtimeForFirst);
            definedTimeLine.setDateToForFirst(closingtimeForFirst);
            definedTimeLine.setDateFromForSecond(startingtimeForSecond);
            definedTimeLine.setDateToForSecond(closingtimeForSecond);

            if (isClocking == EAttendanceClockRule.TWOCLOCK.getValue()) {
                definedTimeLine.setClockType(ClockType.TwoClockPerDay);
            } else if (isClocking == EAttendanceClockRule.FOURCLOCK.getValue()) {
                //四卡的时候，需要设置最晚签退和最早签到
                definedTimeLine.setClockType(ClockType.FourClockPerDay);
                List<AttendanceClockTimeRangeSettingPO> timeRanges = clockSituationDO.getClockSettingPO().getTimeRanges();
                AttendanceClockTimeRangeSettingPO timeRangeSettingPOMorning = timeRanges.get(0);
                AttendanceClockTimeRangeSettingPO timeRangeSettingPOAfternoon = timeRanges.get(1);
                String clockEndTimeMorning = timeRangeSettingPOMorning.getClockEndTime();//上午最晚签退
                String clockEndTimeAfternoon = timeRangeSettingPOAfternoon.getClockStartTime();//下午最早签退
                definedTimeLine.setLatestReturningDate(clockEndTimeMorning);
                definedTimeLine.setLatestSigningDate(clockEndTimeAfternoon);
            }
            //
            AttendancePlanSettingPO planSettingPO = clockSituationDO.getPlanSettingPO();
            if (null != planSettingPO && planType == PlanType.SCHEDULING) {
                if (planSettingPO.getPayType() == EAttendancePayType.LEGAL_WORKING_DAY.getValue()
                        && planSettingPO.getPayAttendanceType() == EPayAttendanceType.ATTENDANCE_TYPE_HOUR.getValue()
                        && planSettingPO.getDayLength() > 0) {
                    BigDecimal b = new BigDecimal(String.valueOf(planSettingPO.getDayLength()));
                    definedTimeLine.setWorkHoursOneDay(b.toPlainString());
                }
            }
            definedTimeLine.setIsWorkDay(isworkDay);
            int type = clockSituationDO.getClockSettingPO().getType();
            /**
             * public enum EAttendanceClockType {
             *     ONANDOFF(1, "上下班打卡"),//上下班打卡，豁免时间
             *     DURATION(2, "时长打卡"),//时长打卡  弹性时间
             *     NOCLOCK(3, "不打卡");//不打卡
             */
            definedTimeLine.setFlexMinutes(0).setExemptionMinutes(0);

            if (type == EAttendanceClockType.DURATION.getValue()) {//弹 性
                definedTimeLine.setFlexMinutes(clockSituationDO.getClockSettingPO().getFlextime());
            } else if (type == EAttendanceClockType.ONANDOFF.getValue()) {//火棉
                definedTimeLine.setExemptionMinutes(clockSituationDO.getClockSettingPO().getFlextime());
            }
            //晚走晚到 排班没有晚走晚到，
            List<LateLeaveRangeSettingDO> lateArriveSettings = planSettingPO.getLateArriveSettings();
            if (lateArriveSettings != null) {
                List<LeavingLateAndComingLateItemRule> collect = lateArriveSettings.stream().filter(p -> p.getLateLeaveSettingType() == ELateLeaveSettingType
                        .LATE_LEAVE_LATE_ARRIVE.getValue()).map(p -> new LeavingLateAndComingLateItemRule().setFlagDateString(p.getLimitedTime()).setLeavingWhen(p.getTime())
                        .setComingLateIn
                                (String.valueOf(p.getLateArriveHour()))).collect(Collectors.toList());

                if (collect != null && collect.size() > 0) {
                    if (clockSituationDO.getYesterdaySignOut() != null) {
                        LeavingLateAndComingLateRule leavingLateAndComingLateRule = new LeavingLateAndComingLateRule().setItemRules(collect).setYesterdayClockTime
                                (clockSituationDO.getYesterdaySignOut().getClockTime());
                        definedTimeLine.getProperties().put("leavingLateAndComingLateRule", leavingLateAndComingLateRule);
                    }
                }
            }
        }

        //没有休息时间,且getTimeRanges()==1，为只有一段设置
        if (clockSettingPO.getTimeRanges().size() == 1) {
            if (definedTimeLine.getDateToForFirst().equals(definedTimeLine.getDateFromForSecond())) {
                String dateFromForFirst = definedTimeLine.getDateFromForFirst();
                String[] split = dateFromForFirst.split(":");

                String dateToForSecond = definedTimeLine.getDateToForSecond();
                String[] split1 = dateToForSecond.split(":");

                String dateFromForFirstHour = split[0];
                String dateToForSecondHour = split1[0];
                int dateFromForFirstHourInt = Integer.valueOf(dateFromForFirstHour);
                int dateToForSecondHourInt = Integer.valueOf(dateToForSecondHour);
                Date timeAddMinute = DateUtil.getTimeAddMinute(definedTimeLine.getShouldClockTimeForFirstBeginOnSetting(), 10);

                if (definedTimeLine.getExemptionMinutes() > 0) {
                    timeAddMinute = DateUtil.getTimeAddMinute(timeAddMinute, definedTimeLine.getExemptionMinutes());

                } else if (definedTimeLine.getFlexMinutes() > 0) {
                    timeAddMinute = DateUtil.getTimeAddMinute(timeAddMinute, definedTimeLine.getFlexMinutes());
                }

                if (timeAddMinute.getTime() > definedTimeLine.getShouldClockTimeForSecondEndOnSetting().getTime()) {
                    timeAddMinute = DateUtil.getTimeAddMinute(definedTimeLine.getShouldClockTimeForSecondEndOnSetting(), -10);
                }
                String hoursOfDate = DateUtil.getHoursOfDate(timeAddMinute);
                String minuteOfDate = DateUtil.getMinuteOfDate(timeAddMinute);
                definedTimeLine.setDateToForFirst(hoursOfDate + ":" + minuteOfDate);
                definedTimeLine.setDateFromForSecond(hoursOfDate + ":" + minuteOfDate);
            }
            definedTimeLine.setNoneRest(true);
        }
```

###### 2、组装DataKey key=companyId-employeeId-date

```java
DataKey dataKey = DataKey.from(companyId, employeeId, new Date(attendanceDate.getTime()));
```

```java
public class DataKey {

    private Date date;

    private MainId mainId;

    private String key;

    public static DataKey from(MainId mainId, Date date) {
        //DateFormat formatUpperCase = new SimpleDateFormat("YYYY-MM-dd");

        String stringDateShort = DateUtil.getStringDateShort(date);

        return new DataKey().setDate(date).setMainId(mainId).setKey(String.format("%s-%s-%s",
                mainId.getCompanyId(), mainId.getEmployeeId(), stringDateShort));
    }

    public static DataKey from(String company, String employeeId, Date date) {
        return DataKey.from(new MainId().setCompanyId(company).setEmployeeId(employeeId), date);
    }
  }


public class MainId {

    String companyId;
    String employeeId;
  }
```

###### 3、调用yank模块

```java
List<AttendanceDailyStatusDetailPO> attendanceDailyStatusDetailPOS1 = yankClient.fireWithResult(dataKey, definedTimeLine);
```

###### 4、更新attendance_daily_status_detail表

```java
List<AttendanceDailyStatusDetailPO> ados = refreshModel.getDailyStatusDetailPOList();
iAttendanceDailyStatusDetailInCommonInCommon.updateDailyStatusDetailByLock(recordPO.getId(), ados);
```

###### 5、更新attendance_month_brief表

```java
if (recordPO != null) {
    AttendanceArchivePO archivePO = iAttendanceArchiveServiceInCommon.getActiveArchive(recordPO.getCompanyId());
    archivePO = iAttendanceArchiveServiceInCommon.getArchiveByCompanyIdAndDate(recordPO.getCompanyId(),
            DateUtil.getTimesByAttendanceDate(recordPO.getAttendanceDate()), archivePO);
    iAttendanceMonthBriefService.updateBrief(recordPO.getCompanyId(), recordPO.getEmployeeId(), archivePO);
}
```

###### 6、发送TransactionMessageEvent事件

```java
if (planSettingPO.getPlanType() == EAttendancePlanType.SCHEDULING.getValue()) {
      publisher.publishEvent(new TransactionMessageEvent(this, recordPO.getCompanyId(), recordPO.getEmployeeId(), recordPO.getAttendanceDate()));
    }
```


###### 7、发送假勤固化事件

```java
if (planSettingPO.getPlanType() == EAttendancePlanType.SCHEDULING.getValue()) {
  publisher.publishEvent(new TransactionMessageEvent(this, recordPO.getCompanyId(), recordPO.getEmployeeId(), recordPO.getAttendanceDate()));
}
```

###### 8、数据改动事件

```java
if (refreshStat) {
                if (changedInfos.size() > 0) {
                    Map<String, List<DailyStatusDetailChangedInfo>> collect = changedInfos.stream().collect
                            (Collectors.groupingBy(p -> p.getCompanyId()));

                    for (String _companyId : collect.keySet()) {
                        List<DailyStatusDetailChangedInfo> changedInfos1 = collect.get(_companyId);
                        SyncEngineMqMsgEvent syncEngineMqMsgEvent = situationGeneratorForEngine
                                .generateSyncEngineMqMsgEvent(_companyId, changedInfos1);
                        //保存排班时，原有逻辑一共调用了3次固化信息，现改为调用一次，只有整体处完排班后，调用固化逻辑，所以以下暂时注释掉
                        publisher.publishEvent(syncEngineMqMsgEvent);
                    }
                }
            }
```
