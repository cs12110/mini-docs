# mybatis-plus

All about mybatis plus, u kown. :"}

---

### 1. saveBatch

Q: 因为项目规范里面,service 不调用 service,如果想用 mybatis plus 的 saveBatch 好像就没办法了?

A: 之前我也是这么以为的.那是不是可以抄袭一下 serviceImpl 的 saveBatch ? 打字人的事算抄么???

```java
/**
 * 批量保存数据,节省时间
 *
 * @param orderItems {@link  UserOrderItemDO}
 */
private void saveBatchUserOrderItem(List<UserOrderItemDO> orderItems) {
    if (ListUtils.isEmpty(orderItems)) {
        return;
    }
    Log logger = LogFactory.getLog(this.getClass());
    String sqlStatement = SqlHelper.getSqlStatement(UserOrderItemMapper.class, SqlMethod.INSERT_ONE);

    SqlHelper.executeBatch(UserOrderItemDO.class, logger, orderItems, orderItems.size(),
                           (sqlSession, entity) -> sqlSession.insert(sqlStatement, entity));
}
```

---

### 2. 指定字段

Q: 在 mybatis-plus 里面执行查询,默认都是`select * from`,那么有没有办法减少不必要的字段呀?

A: 可以使用如下范例:

```java
public List<OcrInfo> getSomeFields(String md5) {
    LambdaQueryWrapper<OcrInfo> queryWrapper = new LambdaQueryWrapper<>();
    queryWrapper
            // 设置只获取出来的字段
            .select(OcrInfo::getMd5, OcrInfo::getOcrChannel)
            // 设置查询条件
            .eq(OcrInfo::getMd5, md5);

    return baseMapper.selectList(queryWrapper);
}
```

---

### 3. 复杂查询语句

Q: 在`LambdaQueryWrapper`怎么构建复杂查询 sql 呀?

A: 比如:`WHERE ( (c1=a AND c2=b ) OR (c1=c AND c2=d) )`.

```java
LambdaQueryWrapper<InvoiceInfo> queryWrapper = new LambdaQueryWrapper<>();
        queryWrapper.and(wrapper -> {
            for (SettlementInvoiceInfoResp each : respValues) {
                wrapper.or((orw) -> {
                    orw.eq(InvoiceInfo::getInvoiceNo, each.getInvoiceNo())
                            .eq(InvoiceInfo::getInvoiceCode, each.getInvoiceCode());
                });
            }
        });
```

A: 比如:`WHERE 1=1 AND ( c1=a OR c2=b AND c3=d)`.

```java
/**
 * 加载审批单数据
 *
 * @param operatorId 用户Id
 * @param billType   单据类型
 * @param endDate    结束时间
 * @param startDate  开始时间
 * @return List
 */
private List<BillInfo> loadBillInfos(String operatorId,
                                     BillType billType,
                                     LocalDateTime endDate,
                                     LocalDateTime startDate) {
    // 使用and(condition1 or condition2 or condition3)条件来查询
    LambdaQueryWrapper<BillInfo> billQueryWrapper = Wrappers.<BillInfo>lambdaQuery()
            .eq(BillInfo::getStaffNo, operatorId)
            .eq(BillInfo::getBillType, billType)
            .and((wrapper) -> {
                // 审批通过
                wrapper.or((sub)->
                    sub.le(BillInfo::getApprovedDate, endDate).ge(BillInfo::getApprovedDate, startDate)
                );
                // 审批中
                wrapper.or(sub->{
                    sub.le(BillInfo::getSubmitDate, endDate).ge(BillInfo::getSubmitDate, startDate);
                });
                // 驳回
                wrapper.or(sub->{
                    sub.le(BillInfo::getRejectDate, endDate).ge(BillInfo::getRejectDate, startDate);
                });
            });

    return baseMapper.selectList(billQueryWrapper);
}
```

---

### 4. sql 日志

Q: 在日常会遇到 sql 打印太多,追踪起来很麻烦.那么有没有动态控制 sql 打印的东西呀?在不重启项目的情况下,需要打印 sql 则开启,不需要则关闭?

A: 在追踪源码里面,可以看出 sql 打印是 mappedstatement 里面的`statementLog`. [详情 blog link](https://juejin.cn/post/6997724404989820964)

```java
package com.pkgs.controller.mock;

import cn.hutool.core.collection.CollUtil;
import com.alibaba.fastjson.JSON;
import com.baomidou.mybatisplus.autoconfigure.MybatisPlusProperties;
import com.baomidou.mybatisplus.core.MybatisConfiguration;
import com.pkgs.entity.SettlementInfo;
import com.pkgs.model.Result;
import com.pkgs.service.SettlementInfoService;
import io.swagger.annotations.ApiOperation;
import lombok.Data;
import lombok.extern.slf4j.Slf4j;
import org.apache.ibatis.logging.Log;
import org.apache.ibatis.logging.nologging.NoLoggingImpl;
import org.apache.ibatis.logging.stdout.StdOutImpl;
import org.apache.ibatis.mapping.MappedStatement;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;
import springfox.documentation.annotations.ApiIgnore;

import javax.annotation.Resource;
import java.lang.reflect.Field;
import java.util.ArrayList;
import java.util.Collection;
import java.util.List;
import java.util.Objects;

/**
 * 终于实现了,泪目.
 *
 * @author huapeng.huang
 * @version V1.0
 * @since 2022-05-17 15:44
 */
@Slf4j
@RestController
@RequestMapping("/mock/mybatis")
@ApiOperation(value = "mybatis", hidden = true)
@ApiIgnore
public class MybatisController {

    @Resource
    private MybatisPlusProperties mybatisPlusProperties;

    @Resource
    private SettlementInfoService settlementInfoService;

    @Data
    public static class DynamicArgs {
        /**
         * mapper数据,如果为空,则设置全部mapper
         */
        private List<String> mapperList;

    }

    @PostMapping("/open")
    public Result<?> open(@RequestBody DynamicArgs dynamicArgs) {
        refreshStatementLog(dynamicArgs.getMapperList(), Boolean.TRUE);

        // 检测数据sql打印是否正常开启/关闭
        SettlementInfo before = settlementInfoService.getBySettlementNo("123");
        log.info("Function[mybatis] value:{}", JSON.toJSONString(before));

        return Result.success(before);
    }

    @PostMapping("/close")
    public Result<?> close(@RequestBody DynamicArgs dynamicArgs) {
        refreshStatementLog(dynamicArgs.getMapperList(), Boolean.FALSE);

        // 检测数据sql打印是否正常开启/关闭
        SettlementInfo before = settlementInfoService.getBySettlementNo("123");
        log.info("Function[mybatis] value:{}", JSON.toJSONString(before));

        return Result.success(before);
    }

    /**
     * 刷新mapper的日志
     *
     * @param mapperList mapperList
     * @param isOpen     isOpen
     */
    private void refreshStatementLog(List<String> mapperList, boolean isOpen) {
        // 获取所有mybatis处理的mapper
        MybatisConfiguration configuration = mybatisPlusProperties.getConfiguration();
        Collection<MappedStatement> mappedStatements = configuration.getMappedStatements();

        StdOutImpl stdOut = new StdOutImpl(MybatisController.class.getName());
        NoLoggingImpl noLogging = new NoLoggingImpl(MybatisController.class.getName());
        Log settlementLog = isOpen ? stdOut : noLogging;

        String mapperId = null;
        List<String> mapperIds = new ArrayList<>();
        for (Object mt : mappedStatements) {
            // 判断是否属于MappedStatement
            boolean isMappedStm = (mt instanceof MappedStatement);
            if (!isMappedStm) {
                continue;
            }

            // 设置settlementLog,如果不为null,则设置对应的mapper
            MappedStatement mst = (MappedStatement) mt;
            if (CollUtil.isNotEmpty(mapperList)) {
                mapperId = getMapper(mst);
                if (!mapperList.contains(mapperId)) {
                    continue;
                }
            }


            try {
                Field settlementLogField = mst.getClass().getDeclaredField("statementLog");
                settlementLogField.setAccessible(true);
                settlementLogField.set(mst, settlementLog);

                mapperIds.add(mapperId);
            } catch (Exception e) {
                log.warn("Function[refreshStatementLog] mapperId:" + mapperId, e);
            }
        }
        log.info("Function[refreshStatementLog] class:{}, mapperIds:{}",
                settlementLog.getClass(),
                String.join(",", mapperIds)
        );
    }


    private String getMapper(MappedStatement mst) {
        if (Objects.isNull(mst)) {
            return null;
        }
        // mapperClass.method
        String id = mst.getId();
        int last = id.lastIndexOf(".");
        return id.substring(0, last);
    }
}
```

---

### 参考文档

a. [mybatis-plus 官方文档](https://baomidou.com/pages/24112f/)

b. [mybatis 打印完整 sql 日志](https://juejin.cn/post/6997724404989820964)
