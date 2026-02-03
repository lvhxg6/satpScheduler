# 任务自动下发配置 - 补充内容

本文档记录开发过程中需要参考的SQL查询和接口调用示例。

---

## 1. 工具查询

### 1.1 根据专题查询对应的工具

**主表名称**：`RPA_ABILITY_BASIC`  
**对应实体类**：`AbilityBasic`

```java
public List<AbilityBasic> queryAbilityBasicList(String typeCode) {
    StringBuilder sql = new StringBuilder();
    List<Object> params = new ArrayList<Object>();

    sql.append("select distinct rab.*,rat.type_name,rat.type_code,agv.VER_CODE,agr.PK_ABILITY_GA_RE ability_version_id,\n" +
            "ap.PROVIDER_ICON\n" +
            "from RPA_ABILITY_BASIC rab\n" +
            "left join RPA_ABILITY_TYPE rat on rab.pk_ability_type=rat.pk_ability_type\n" +
            "left join RPA_ABILITY_GA_RE agr on agr.pk_ability_basic=rab.pk_ability_basic\n" +
            "left join RPA_ABILITY_GA_VER agv on agv.PK_ABILITY_GA_VER=agr.PK_ABILITY_GA_VER\n" +
            "left join RPA_ABILITY_GA ag on ag.PK_ABILITY_GA = agv.PK_ABILITY_GA\n" +
            "left join RPA_ABILITY_PROVIDER ap on ap.PK_ABILITY_PROVIDER = ag.PK_ABILITY_PROVIDER\n" +
            "left join RPA_ABILITY_ACCESS_RE raar on raar.PK_ABILITY_GA_RE=agr.PK_ABILITY_GA_RE\n" +
            "right join rpa_ability_access raa on raa.PK_ABILITY_ACCESS=raar.PK_ABILITY_ACCESS\n" +
            "where rat.type_code = ? and rab.is_deleted = 0\n" +
            "and raa.IS_DELETED = 0 and raa.ABILITY_STATUS = 2 and raar.SRV_STATUS = 2 " +
            "and rab.ability_code not in('PX_CLOUD_CONFIG_CHECK','PX_CLOUD_PWD_CHECK') ").append(LS);

    params.add(typeCode);

    sql.append(" order by rab.DISPLAY_SEQ asc ");
    return super.queryBySql(sql.toString(), params.toArray());
}
```

**返回JSON示例**：

```json
[
    {
        "props": {
            "abilityVersionId": "7e54aa70c829421ab6c3c9a90a4aa85e",
            "typeName": "基线合规在线检测",
            "providerIcon": "bms",
            "typeCode": "CONFIG_CHECK",
            "verCode": "V1.0"
        },
        "serNum": null,
        "key": null,
        "name": null,
        "pkAbilityBasic": "d52b548989f44543a0b373621a5cdfec",
        "abilityName": "泰岳基线合规在线检测",
        "abilityCode": "ULTRA_CONFIG_CHECK",
        "pkAbilityType": "0f2ca2d57d0942dd9581959fe835102d",
        "abilityDesc": "{\"maxJobNum\":10,\"jobMaxIpNum\":500}",
        "abilityLogo": null,
        "displaySeq": 9,
        "isDeleted": 0,
        "search": "泰岳基线核查##神州泰岳##Ultra-BMS##V 1.0.11",
        "interfaceClass": "com.rpa.ability.plugin.intef.amr.bms.v1011.ConfigServiceInterface",
        "checkSoFormat": 1,
        "checkSoSource": 0
    }
]
```

### 1.2 四个专题的编码

| 专题 | 编码 |
|------|------|
| 基线 | CONFIG_CHECK |
| 弱口令 | PWD_CHECK |
| 系统漏洞 | SYS_VULN |
| WEB漏洞 | WEB_VULN |

---

## 2. 模板查询

### 2.1 根据工具ID查询对应的模板信息

```sql
SELECT foreign_template_id, foreign_template_name 
FROM view_ability_template 
WHERE pk_ability_basic = 'd52b548989f44543a0b373621a5cdfec';
```

---

## 3. 资产查询

### 3.1 是否上报筛选条件

```java
/** 是否上报 0 否 1 是 是否上报 REPORT_STATUS */
if (!nullOrEmpty(aqv.getReportStatus())) {
    sql.append("AND EXISTS (SELECT 1 FROM g_asset_extend_prop S " +
               "WHERE S.prop_code='REPORT_STATUS' and s.prop_value=? " +
               "and S.PK_ASSET = CA.PK_ASSET) ").append(LS);
    params.add(aqv.getSopropDomain());
}
```

### 3.2 四个专题查询资产的SQL

**系统漏洞**：

```sql
SELECT ga.* 
FROM g_asset ga
LEFT JOIN g_asset_type gat ON ga.asset_type_id = gat.pk_asset_type
WHERE ga.is_deleted = 0 
  AND (gat.type_code LIKE 'OS_%' 
       OR gat.type_code LIKE 'NETDEV_%' 
       OR gat.type_code LIKE 'SECDEV_%')
```

**WEB漏洞**：

```sql
SELECT ga.* 
FROM g_asset ga
LEFT JOIN g_asset_type gat ON ga.asset_type_id = gat.pk_asset_type
WHERE ga.is_deleted = 0 
  AND gat.type_code = 'APP'
```

**基线**：

```sql
SELECT ga.* 
FROM g_asset ga
WHERE ga.is_deleted = 0 
  AND EXISTS (
      SELECT 1 
      FROM (
          SELECT bcr.pk_so 
          FROM bd_checkitem bc
          LEFT JOIN bd_checkitem_related bcr ON bc.pk_checkitem = bcr.pk_checkitem 
          WHERE bc.property = 0 AND bc.is_deleted = 0
      ) t 
      WHERE t.pk_so = ga.asset_type_id
  )
```

**弱口令**：

```sql
SELECT ga.* 
FROM g_asset ga
WHERE ga.is_deleted = 0 
  AND EXISTS (
      SELECT 1 
      FROM (
          SELECT bcr.pk_so 
          FROM bd_checkitem bc
          LEFT JOIN bd_checkitem_related bcr ON bc.pk_checkitem = bcr.pk_checkitem 
          WHERE bc.property = 9 AND bc.is_deleted = 0
      ) t 
      WHERE t.pk_so = ga.asset_type_id
  )
```

---

## 4. AMR接口

### 4.1 接口定义

```java
@FeignClient(name = "https://amr-webapi", fallbackFactory = AmrTaskClientFallback.class)
public interface AmrTaskClient {

    /**
     * 应用端下发综合检查脚本
     */
    @PostMapping("/v1/thirdSecurity/create")
    Result<AmrTaskCreateResponseVo> taskCreate(@RequestBody AmrTaskCreateVo checkPlan);
}
```

### 4.2 任务创建调用示例

```java
public GroupCreateTaskVo taskCreate(String taskId) {
    BMSGroupParamInfo bmsGroupParamInfo = bmsGroupParamInfoDAO.queryById(taskId);
    if (Objects.isNull(bmsGroupParamInfo)) {
        throw new BizIllegalArgumentException("未查询到集团检查任务");
    }
    if (BMSGroupParamInfo.GroupTaskState.SOBIND.getValue() != bmsGroupParamInfo.getSoState()) {
        throw new BizIllegalArgumentException("集团任务资产匹配未提交，请提交之后再发起任务");
    }
    
    List<BmsGroupAsset> groupAssets = bmsGroupAssetDAO.queryTaskId(taskId);
    if (CollectionUtils.isEmpty(groupAssets)) {
        throw new BizIllegalArgumentException("集团任务资产全部取消，无法发起核查任务");
    }
    logger.debug("符合发起任务条件资产个数：" + groupAssets.size());
    
    AmrTaskCreateVo resolve = resolve(bmsGroupParamInfo, groupAssets);
    logger.debug("调用AMR创建任务结果请求：" + JsonUtil.toJSON(resolve));
    
    Result<AmrTaskCreateResponseVo> taskCreateResult = amrTaskClient.taskCreate(resolve);
    logger.debug("调用AMR创建任务返回结果：" + JsonUtil.toJSON(taskCreateResult));
    
    if (!taskCreateResult.isSuccess()) {
        throw new BizServiceException(taskCreateResult.getMessage());
    }
    return new GroupCreateTaskVo(resolve.getTaskName());
}
```

### 4.3 构建任务创建请求对象

```java
/**
 * 构建任务创建发起对象
 */
private AmrTaskCreateVo resolve(BMSGroupParamInfo bmsGroupParamInfo,
                                List<BmsGroupAsset> groupAssets) {
    AmrTaskCreateVo createVo = new AmrTaskCreateVo();
    createVo.setTaskName("集团巡检任务_" + bmsGroupParamInfo.getTaskName());
    createVo.setTaskClass(String.valueOf(bmsGroupParamInfo.resolveAmrTaskType()));
    createVo.setPlanSource(String.valueOf(4)); // 安全运营平台
    createVo.setTargetList(resolveTargetAsset(groupAssets));
    createVo.setAbilityList(resolveAbility(bmsGroupParamInfo));
    createVo.setTaskExtendMap(resolveTaskExtendMap());
    return createVo;
}

/**
 * 构建资产列表
 */
private List<AmrTaskCreateVo.Asset> resolveTargetAsset(List<BmsGroupAsset> groupAssets) {
    List<AmrTaskCreateVo.Asset> assets = new ArrayList<>();
    for (BmsGroupAsset ga : groupAssets) {
        AmrTaskCreateVo.Asset a = new AmrTaskCreateVo.Asset();
        a.setAssetId(ga.getAssetId());
        assets.add(a);
    }
    return assets;
}

/**
 * 构建能力对象
 */
private List<AmrTaskCreateVo.Ability> resolveAbility(BMSGroupParamInfo bmsGroupParamInfo) {
    List<AmrTaskCreateVo.Ability> abilities = new ArrayList<>();
    AmrTaskCreateVo.Ability ability = new AmrTaskCreateVo.Ability();
    ability.setAbilityCode(bmsGroupParamInfo.resolveAmrAbilityCode());
    ability.setAbilityVer(resolveAbilityVer(bmsGroupParamInfo));
    ability.setScanParams(resolveScanParam(bmsGroupParamInfo));
    abilities.add(ability);
    return abilities;
}

/**
 * 获取能力版本
 */
private String resolveAbilityVer(BMSGroupParamInfo bmsGroupParamInfo) {
    AmrAbility amrAbility = abilityDAO.queryByCode(bmsGroupParamInfo.resolveAmrAbilityCode());
    if (Objects.isNull(amrAbility)) {
        throw new BizServiceException("未查询到可用工具，请先确认是否存在可用工具");
    }
    return String.valueOf(amrAbility.getProps().get("verCode"));
}

/**
 * 分专题构建扫描参数
 */
private AmrTaskCreateVo.ScanParam resolveScanParam(BMSGroupParamInfo bmsGroupParamInfo) {
    AmrTaskCreateVo.ScanParam param = new AmrTaskCreateVo.ScanParam();
    param.setIsSaveOriginal("1");
    param.setTemplateId(resolveTemplateId(bmsGroupParamInfo));
    return param;
}
```

### 4.4 请求VO定义

```java
@Data
@NoArgsConstructor
public class AmrTaskCreateVo {

    /**
     * 任务类型：基线 16, 弱口令 17, 防火墙 20
     */
    String taskClass;
    String taskDesc;
    List<Asset> targetList;
    List<Ability> abilityList;
    String checkWay;
    String taskName;
    
    /**
     * 计划来源：2-安全运营平台，3-态势
     */
    String planSource;

    /**
     * 计划扩展参数信息
     */
    private Map<String, String> taskExtendMap = new HashMap<>();

    @Data
    @NoArgsConstructor
    public static class Asset {
        String assetId;
    }

    @Data
    @NoArgsConstructor
    @AllArgsConstructor
    public static class Ability {
        String abilityCode;
        String abilityVer;
        ScanParam scanParams;
    }

    @Data
    @NoArgsConstructor
    @AllArgsConstructor
    @JsonInclude(JsonInclude.Include.NON_NULL)
    public static class ScanParam {
        String isSaveOriginal;
        String templateId;
        
        @JsonIgnore
        @Deprecated
        String isGetPwd; // amr接口已经将该字段废弃

        public ScanParam(String isSaveOriginal, String templateId) {
            this.isSaveOriginal = isSaveOriginal;
            this.templateId = templateId;
        }
    }
}
```

### 4.5 响应VO定义

```java
@Data
@NoArgsConstructor
public class AmrTaskCreateResponseVo {

    String taskId;
    String forbidMsg;
    List<UnSupport> unSupportList;

    @Data
    @NoArgsConstructor
    public static class UnSupport {
        String assetId;
        String targetIp;
        String soErrorDesc;
    }
}
```

---

## 5. 任务类型映射

| 专题 | taskClass |
|------|-----------|
| 基线 | 16 |
| 弱口令 | 17 |
| 防火墙 | 20 |
