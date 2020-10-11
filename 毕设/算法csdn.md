# 前言

后期随着系统的不断壮大，用户面对的选择增多，一个智能的推荐算法也称为业务的重点。

本文主要介绍基于物品的协同过滤算法。

```tex
1.基于用户的协同过滤算法（UserCF）

该算法利用用户之间的相似性来推荐用户感兴趣的信息，个人通过合作的机制给予信息相当程度的回应（如评分）并记录下来以达到过滤的目的进而帮助别人筛选信息，回应不一定局限于特别感兴趣的，特别不感兴趣信息的纪录也相当重要。但有很难解决的两个问题，一个是稀疏性，即在系统使用初期由于系统资源还未获得足够多的评价，很难利用这些评价来发现相似的用户。另一个是可扩展性，随着系统用户和资源的增多，系统的性能会越来越差。

2.基于物品的协同过滤算法（ItemCF）

内容过滤根据信息资源与用户兴趣的相似性来推荐商品，通过计算用户兴趣模型和商品特征向量之间的向量相似性，主动将相似度高的商品发送给该模型的客户。由于每个客户都独立操作，拥有独立的特征向量，不需要考虑别的用户的兴趣，不存在评价级别多少的问题，能推荐新的项目或者是冷门的项目。这些优点使得基于内容过滤的推荐系统不受冷启动和稀疏问题的影响。
```

本文以慕课平台为例，业务逻辑主要是：学生选择自己感兴趣的课程，同时，需要通过推荐算法给学生推荐课程。

# 正文

## 算法介绍

**算法流程**

1.构建学生–>课程的倒排；

2.构建课程与课程的同现矩阵；

3.计算课程之间的相似度，即计算相似矩阵；

4.根据学生的历史记录，给学生推荐课程；

### 算法流程1

如下表，行表示学生，列表示课程，1表示学生选择了该课程

| 学生\课程 | a    | b    | c    | d    |
| --------- | ---- | ---- | ---- | ---- |
| A         |      | 1    | 1    |      |
| B         |      | 1    | 1    | 1    |
| C         | 1    |      |      | 1    |

### 算法流程2 

构建课程与课程的同现矩阵。

共现矩阵C表示同时喜欢两个课程的学生数，可以计算出如下的共现矩阵C：

| 课程\课程 | a    | b    | c    | d    |
| --------- | ---- | ---- | ---- | ---- |
| a         |      | 1    |      | 1    |
| b         | 1    |      | 2    |      |
| c         |      | 2    |      | 1    |
| d         | 1    |      | 1    |      |

明显可以看出，该矩阵为对称矩阵。

### 算法流程3

计算课程之间的相似度，即计算相似矩阵。

设|N(i)|表示喜欢课程i的学生数，|N(i)⋂N(j)|表示同时喜欢课程i，j的学生数，则课程i与课程j的相似度为： 

wij=|N(i)⋂N(j)| / |N(i)|

(1)式有一个问题，当课程j是一个很热门的课程时，人人都选择，比如学校所有人要求必修的课程，那么wijwij就会很接近于1，即(1)式会让很多课程都和此课程有一个很大的相似度，所以可以改进一下公式：

wij=|N(i)⋂N(j) / √||N(i)||N(j)|

矩阵N如下所示：

| 课程       | a    | b    | c    | d    |
| ---------- | ---- | ---- | ---- | ---- |
| **学生数** | 1    | 2    | 2    | 2    |

其中余弦相似计算如下：

wab=1/√1*2=1/√2=0.71

wbc=2/√2*2=2/2=1

wad=1/√1*2=1/√2=0.71

wcd=1/√2*2=2/2=1

利用式（2）便能计算课程之间的余弦相似矩阵如下：

| 课程\课程 | a    | b    | c    | d    |
| --------- | ---- | ---- | ---- | ---- |
| a         |      | 0.71 |      | 0.71 |
| b         | 0.71 |      | 1    |      |
| c         |      | 1    |      | 1    |
| d         | 0.71 |      | 1    |      |

### 算法流程4

课程j预测兴趣度=学生选择的课程i的兴趣度×课程i和课程j的相似度

例如：A学生选择课程b，c ，令兴趣度均为1

- 学生A对课程a的预测兴趣度=课程b兴趣度 * 课程ab的相似度+课程c兴趣度 * 课程ac的相似度=1 * 0.71 + 1 * 0 =0.71
- 学生A对课程d的预测兴趣度=课程b兴趣度 * 课程db的相似度+课程c兴趣度 * 课程dc的相似度=1 * 0 + 1 * 1=1
- 因此，对于学生A，课程推荐顺序为：d>c

当然，以上数据量较小，课程增多后，将可以按照学生A对各课程的兴趣度进行排序，选出推荐的m个课程。

## 数据库设计

用户课程表t_user_course，有如下字段：

* id
* user_id
* course_id

课程表，有如下字段：

* id
* course_name

用户表，有如下字段：

* id
* user_name

课程权重表，有如下字段：

* id（自增，方便批量插入）
* row_course_id
* col_course_id
* point

注意：课程权重表即相当于以上的余弦相似矩阵。

## 代码实现

当课程数量、用户选课数量增多时，以上算法是十分耗时的，所以需要每天定时任务更新课程权重，插入数据库，提供效率。

### utils类

首先有RecommendDto类，方便以下计算

```java
/**
 * @author hpsyche
 * Create on 2020/3/28
 */
@Data
@AllArgsConstructor
public class RecommendDto {
    private Long courseIdRows;
    private Long courseIdCols;

    /**
    * 重写equals，方便推荐算法计算
    */
    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        RecommendDto that = (RecommendDto) o;
        return Objects.equals(courseIdRows, that.courseIdRows) &&
                Objects.equals(courseIdCols, that.courseIdCols);
    }

    @Override
    public int hashCode() {
        return Objects.hash(courseIdRows, courseIdCols);
    }
}

```

有UserCourse类，如下

```java
/**
 * @author hpsyche
 * Create on 2019/12/11
 */
@Data
@Table(name = "t_user_course")
public class UserCourse {
    private Long id;
    private Long userId;
    private Long courseId;
}
```



### 更新相似度矩阵

更新相似度矩阵的算法如下：

```java
/**
 * @author hpsyche
 * Create on 2020/3/28
 */
@Autowired
private RecommendWeightMapper recommendWeightMapper;
@Autowired
private UserCourseMapper userCourseMapper;
@Autowired
private CourseMapper courseMapper;
@Override
public int updateRecommend() {
    //1.删除所有数据
    recommendWeightMapper.deleteAll();
    //2.获取基础数据
    List<Course> courseList = courseMapper.selectAll();
    List<UserCourse> userCourses = userCourseMapper.selectAll();
    //3.计算共线矩阵
    Map<RecommendDto,Integer> map=new HashMap<>();
    for(Course course:courseList){
        List<Long> userIds = userCourses.stream().filter(userCourse -> userCourse.getCourseId().equals(course.getId())).
            map(UserCourse::getUserId).collect(Collectors.toList());
        for(Long userId:userIds){
            List<Long> courseIds = userCourses.stream().filter(userCourse -> userCourse.getUserId().equals(userId))
                .map(UserCourse::getCourseId).collect(Collectors.toList());
            for(Long courseId:courseIds){
                if(!courseId.equals(course.getId())) {
                    RecommendDto recommendDto = new RecommendDto(course.getId(), courseId);
                    if (map.containsKey(recommendDto)) {
                        map.put(recommendDto, map.get(recommendDto) + 1);
                    } else {
                        map.put(recommendDto, 1);
                    }
                }
            }
        }
    }
    //4.计算N矩阵
    Map<Long, Long> mapN = userCourses.stream().collect(Collectors.groupingBy(UserCourse::getCourseId, Collectors.counting()));

    List<RecommendWeight> list=new ArrayList<>();
    //5.计算权重
    Set<RecommendDto> keySet = map.keySet();
    for(RecommendDto dto:keySet){
        //row与col课程的相似度
        double weight=map.get(dto)/Math.sqrt(mapN.get(dto.getCourseIdRows())*mapN.get(dto.getCourseIdCols()));
        RecommendWeight recommendWeight=new RecommendWeight();
        recommendWeight.setRowCourseId(dto.getCourseIdRows());
        recommendWeight.setColCourseId(dto.getCourseIdCols());
        //weight为string类型
        recommendWeight.setPoint(weight+"");
        list.add(recommendWeight);
    }
    //6.批量插入
    return recommendWeightMapper.insertBatch(list);
}
```

```mapper
    <insert id="insertBatch" parameterType="java.util.List">
        insert into t_recommend_weight(row_course_id,col_course_id,point)
        values
        <foreach collection="list" item="item" index="index" separator=",">
            (
                #{item.rowCourseId},
                #{item.colCourseId},
                #{item.point}
            )
        </foreach>
    </insert>
    
    <delete id="deleteAll">
        DELETE
        FROM t_recommend_weight
    </delete>
```

通过以上步骤，相似度矩阵生成并插入数据库。

### 用户推荐

当用户访问"推荐接口"时，需要计算生成最佳课程，并返回课程信息。

注意：由于设计数据量较大，第一次查询后可将课程保存至redis，之后再从redis取，在每次定时任务更新权重后再删除redis缓存。

由于这里主讲算法，redis判断的过程不做演示。

```java
public List<Course> getRecommendCourse(Long studentId) {
    return courseMapper.getRecommendCourse(studentId);
}
```

主要的是mapper中sql语句的编写，如下：

```sql
        select id,name
        from t_course
        where id in
        (
            select row_course_id
            from t_recommend_weight trw
            where col_course_id in
            (
                select tuc.course_id
                from t_user_course tuc
                where tuc.user_id=#{id}
            )
            and row_course_id in
            (
                select id
                from t_course tc
                where id not in
                (
                    select tuc.course_id
                    from t_user_course tuc
                    where tuc.user_id=#{id}
                )
            )
            group by row_course_id
            order by `point` desc
        )
        limit 0,3
```

通过以上方式，即可查询到某学生的最佳3个推荐课程。

# 总结

由于博主才疏学浅，这里只给了较为简单协同过滤算法，且算法的复杂度也较高，可能效率不是很高，有什么不足之处，还望指出。