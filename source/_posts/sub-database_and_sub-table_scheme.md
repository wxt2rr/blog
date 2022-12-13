---
title: 代码实现单表的分库分表
date: 2022-12-04 14:38
categories:
- [思考总结, 数据库优化]
tags:
- 分库分表
- 性能优化
---
线上业务上由于单表的数据量越来越大，并且随着用户的增长，表数据量的增长率可能还会增高，所以想着对单表进行分表，然后将整个分表的缘由以及分表方案的确定记录一下
<!--more-->
### 啥叫分库分表？

简单点说就是当我们遇到访问极为频繁且数据量巨大的库表的时候，我们首先想到的就是减少单库单表的数据量，以便减少数据查询所需要的时间，提高数据库的吞吐，这就是所谓的分库分表。

### 为什么要都开始考虑起来分库分表？


当我们在开发一些用户量小的系统时，比如公司自用的后台系统，相对是比较轻松的，因为你无需特别专注的考虑高并发，高访问，大数据量等问题，所以对于数据库乃至一些表的设计，对当用户量比较小时还可以跑的通，但是随着量越来越大，动不动就上千万，上亿的数据，光靠加索引或对sql往死里优化还是远远不够的，这时候我们就该考虑是否到了对数据库需要进行分库分表的时候了，当然决定了一定要速战速决，因为数据量增长的速度比你犹豫的速度要快的多。

### 如何进行分库分表？

 对于分库分表，现在基本都是*水平分*和*垂直分*：(1)水平分就是根据一定规则，例如按照时间或业务唯一ID等对数据进行的拆分。比如根据年份来拆分不同的数据库。每个数据库结构一致，但是数据会平均，从而提升性能；(2)
垂直分就是按照业务进行划分，例如将用户相关的放一个库，订单相关的放一个库。

#### 1.分库：

 先来讨论下分库，很多项目一开始基本都是一个库，各种业务的表都在一起，随着表越来越多，变得越来越来不好维护，老表新表掺杂在一起，一些废弃的表也没有及时做清理(我真的要吐了，所以文档是多么的重要)
。对于分库，我现在使用较多的是垂直分，按照业务分，比如我们将和用户端无用的一些用于统计的表单独放在一个库里，这样做好处是这些统计表不会参与用户测相关表的join查询操作，并且对于统计表进行需要进行大量计算，这样其实可以更放心的去查数据算数据不用担心影响到用户测的业务，所以对于类似的表进行垂直分库是完全没有问题的。对于水平分库目前没有实践，只是懂一些原理，就....

简单点说就是当我们遇到访问极为频繁且数据量巨大的库表的时候，我们首先想到的就是减少单库单表的数据量，以便减少数据查询所需要的时间，提高数据库的吞吐，这就是所谓的分库分表。

#### 2.分表：

(1)对于向用户ID这样的数值类型我们可以对要分的表的数量取余(userId % count)来进行分表，根据这个路由策略，可以将记录根据userId路由到不同的表中，达到分表的母的。路由帮助方法示例代码：

```java
    // 分表基数
    private static final Integer COUNT = 20;

    /**
     * @desc 根据用户ID获取分表后缀
     * @author dataozi
     * @date 2020/4/11 15:22
     * @param userId 用户ID
     * @return 表后缀
     * @throws RuntimeException 用户ID参数异常
     */
    public static String getSuffixByUserId(Long userId){
        if(Objects.isNull(userId) || userId <= 0){
            throw new RuntimeException(String.format("用户ID有误 , userId : %s", userId));
        }

        return String.format("_g0_p%s", userId % COUNT);
    }
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)可能大家有疑问，为什么表后缀是"_g0_p0"
，这样目的是为了以后可能会再次进行分表，因为这种方式最大的弊端就是我们需要预先确定下表的数量，但是随着量的增长，及时现在分了，以后还是会有再次达到记录阈值的那天；以后如果需要再次进行分表的话，我可以修改为"_g1"再次实现分表。

(2)上边是定表，不定单表记录量的，我们还可以讨论下定表记录数，不行表的。就是我们规定每张表最大数据量假如为100w，那么当第一张表数据量达到后自动开始向下张表存数据。如何实现呢，其实思想大同小异，我们可以维护一个全局的记录总数，每次获取表后缀时判断是否需要取新的表，路由帮助方法示例代码：

```java
    // 单张表记录的最大数量
    private static final long COUNT = 1000000L;

    /**
     * @desc 获取表的坐标
     * @author dataozi
     * @date 2020/4/11 16:15
     * @param count 当前记录数
     * @return 表坐标
     * @throws RuntimeException 记录数参数异常
     */
    public static int getTableIndexVal(long count){
        if (count <= 0) {
            throw new RuntimeException("记录数有误");
        }
        long index = count % COUNT == 0 ? count / COUNT - 1 : count / COUNT;
        return (int) index;
    }
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

需要注意的是一定要在业务记录量达到阈值之前创建新表，我们可以起个定时任务，在表记录达到90w时就创建下张表，预先创建表。

(3)对于向UUID这样的我们可以取UUID的最后一位或最后两位进行分表，以最后一位举例，我们知道UUID是一个数字和字母组成的伪随机字段串，这样的话最后一位就是0-9a-z(26 + 10)，可以建立36张表，同上，表后缀可以是"_g0_pa"，大家没有问题吧。哈哈，坑来了，强调一下，UUID 是由一组32位数的**16进制数字**所构成，所以不是26个字母只有a-f，所以即使按照这种方式创建了表，也是永远用不到的，所以每个小知识点都要熟透啊，我就是对UUID没怎么真正了解，结果...；话说回来，这样的方式确实表的量是死的，那么怎么也变成稍微活一点呢?

(4)我们取UUID的hashcode，这样我们又得到了一个数值类型的唯一标识，同理我们可以这样做，路由帮助方法示例代码：

```java
    // 分表基数
    private static final Integer COUNT = 20;

    /**
     * @desc 根据uuid获取表后缀
     * @author dataozi
     * @date 2020/4/11 15:41
     * @param uuid uuid
     * @return 表后缀
     * @throws RuntimeException uuid参数异常
     */
    public static String getSuffixByUserId(String uuid){
        if (StringUtils.isBlank(uuid)) {
            throw new RuntimeException("designID 为空");
        }

        // 取uuid的hashcode绝对值对20取模的值
        int code = uuid.toLowerCase().hashCode();
        return String.format("_g0_p%s", Math.abs(code) % COUNT);
    }
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)(5)除了上边这种通过唯一标识进行水平分的，我们还可以通过日期进行分表，比如上边提到的一些统计数据表，表本身没有什么业务逻辑，只需要按照日期定期进行数据统计，那我们通过日期进行分表再好不过了；我们按照季度进行分表，一年4张统计数据表，这样分表最好定个闹铃，每个月提醒建表，或者你一次性建个几十年的，O(
∩_∩)O哈哈~，路由帮助方法示例代码：

```java
    // 临时年份值
    private static int tempYearByYearAndSeason;
    // 临时月份值
    private static Month tempMonthByYearAndSeason;
    // 临时表名称后缀
    private static String tempNameSuffixByYearAndSeason;

    /**
     * @desc 获取表后缀
     * @author dataozi
     * @date 2020/4/11 15:49
     * @param year 年
     * @param month 月
     * @return 表后缀
     */
    public static String calcNameSuffix(int year, Month month) {
        // 计算当前属于那个季度，从0开始
        int quarterCount = 0;
        switch (month) {
            case JANUARY:
            case FEBRUARY:
            case MARCH:
                break;
            case APRIL:
            case MAY:
            case JUNE:
                quarterCount += 1;
                break;
            case JULY:
            case AUGUST:
            case SEPTEMBER:
                quarterCount += 2;
                break;
            case OCTOBER:
            case NOVEMBER:
            case DECEMBER:
                quarterCount += 3;
                break;
            default:
                throw new RuntimeException("system error");
        }
        return String.format("_g%s_p%s", year, quarterCount);
    }

    /*
      设置初始值
     */
    static {
        LocalDate now = LocalDate.now();
        tempNameSuffixByYearAndSeason = calcNameSuffix(now.getYear(), now.getMonth());
        tempYearByYearAndSeason = now.getYear();
        tempMonthByYearAndSeason = now.getMonth();
    }

    /**
     * @desc 获取表后缀
     * @author dataozi
     * @date 2020/4/11 15:51
     * @return 表后缀
     */
    public static String getNameSuffixByYearAndSeason() {
        LocalDate now = LocalDate.now();
        int year = now.getYear();
        Month month = now.getMonth();
        // 当年月份交替时需要重置静态数据
        if (year == tempYearByYearAndSeason && month.equals(tempMonthByYearAndSeason)) {
            return Objects.requireNonNull(tempNameSuffixByYearAndSeason);
        }

        tempNameSuffixByYearAndSeason = calcNameSuffix(year, month);
        tempYearByYearAndSeason = year;
        tempMonthByYearAndSeason = month;
        return tempNameSuffixByYearAndSeason;
    }
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

### 分库分表之后，主键ID该怎么维护？

上边我们通过简单的代码，基本已经实现了分库分表，不要高兴太早，原来单表时，我们是有主键ID的，一般都是自增ID，那么进行分表之后，每张表都会有各自的主键自增ID，那么怎么维护一个全局的主键ID呢，下面我们在讨论一下这个问题：

​      (1)利用数据库本身。我们可以在单独创建一张ID表，每次在新增数据时，先向ID表中插入一条数据，获取最新主键ID之后再向分表中插入业务数据；基本这只是一个思路，面临获取ID的性能、ID表记录会无限多等问题不考虑线上使用

​      (2)利用Redis。利用redis的incr命令，每次在插入数据之前，先从redis获取到id，下次获取自动累计，达到自增的作用。示例代码：

```java
    // redis存储全局自增主键ID的key
    private static final String TABLE_RECORD_COUNT_KEY = "xxx-table.primary-key.record";

    /**
     * @desc 获取主键ID
     * @author dataozi
     * @date 2020/4/11 16:20
     * @return 数量
     */
    public static long incr() {
        // cache是我们封装的jedis帮助类
        if (cache == null) {
            // 这里加锁，防止多个线程同时设置
            synchronized (XXXX.class) {
                if (cache == null) {
                    cache = cacheDef.getCACHE19();
                }
            }
        }
        return cache.incr(TABLE_RECORD_COUNT_KEY);
    }
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

(3)其它一些较成熟的自增id的类库，因为没线上实践过，所以...略过

### 如何对已有表进行分表？

对于一些新表好说，我们可以都设计好了在去上线使用，但是我们实际在分表时，有很多是已经在业务使用的表，已经使用的表如何分表呢？这里讨论的是怎么实现由老表向新表的过渡。

(1)停止服务

告诉用户我们要大更新，晚上几点到几点暂停运营，然后在这期间老表没有了新数据的插入，我们可以很方便将历史数据同步到新表，在部署代码，充分测试上线。看似没有任何问题是吧，最大的问题就是方案本身的问题，我们停止了服务，举个不恰当的例子，你怎么判断你的电脑有没有网，是不是打开百度或google随便搜个啥东西，从另外一个方面体现出人家的服务是多么的稳定，所以只要有其它方案，我们绝不考虑停机去干一些事情。

(2)双向同步

不停机我们怎么实现新老表的平滑过渡呢，我们需要先在老表插入数据的地方同步向新表插入并上线，这样入口打通了，然后是历史数据，在搞个定时任务去同步历史数据，等历史数据同步完之后，新表数据和老表就完全一致了，因为历史数据有了，并且新数据会同步向新老表插入，最后一步就是修改查询，将所有从老表查询的地方改为从新表查询，同时去掉向老表插入数据的代码上线，这样老表没有任何使用的地方，任由处置。
