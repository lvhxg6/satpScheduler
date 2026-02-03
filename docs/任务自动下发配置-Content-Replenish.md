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

### 3.1 资产与资产组（业务系统）的表关系

```sql
SELECT ga.asset_name, gag.* 
FROM g_asset ga 
LEFT JOIN g_asset_assetgroup_rel gasr ON ga.pk_asset = gasr.asset_id AND gasr.group_type = 2
LEFT JOIN g_asset_group gag ON gasr.asset_group_id = gag.pk_group
```

**说明**：
- `g_asset`：资产表
- `g_asset_assetgroup_rel`：资产与资产组关联表，`group_type = 2` 表示业务系统
- `g_asset_group`：资产组表（业务系统）

### 3.2 是否上报筛选条件

```java
/** 是否上报 0 否 1 是 是否上报 REPORT_STATUS */
if (!nullOrEmpty(aqv.getReportStatus())) {
    sql.append("AND EXISTS (SELECT 1 FROM g_asset_extend_prop S " +
               "WHERE S.prop_code='REPORT_STATUS' and s.prop_value=? " +
               "and S.PK_ASSET = CA.PK_ASSET) ").append(LS);
    params.add(aqv.getSopropDomain());
}
```

### 3.3 四个专题查询资产的SQL

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

---

## 6. 相关表结构汇总

### 6.1 工具相关表（8张）

| 表名 | 说明 |
|------|------|
| `RPA_ABILITY_BASIC` | 能力/工具基础表（主表） |
| `RPA_ABILITY_TYPE` | 能力类型表（专题分类） |
| `RPA_ABILITY_GA_RE` | 能力与版本关联表 |
| `RPA_ABILITY_GA_VER` | 能力版本表 |
| `RPA_ABILITY_GA` | 能力GA表 |
| `RPA_ABILITY_PROVIDER` | 能力提供商表 |
| `RPA_ABILITY_ACCESS_RE` | 能力接入关联表 |
| `RPA_ABILITY_ACCESS` | 能力接入表 |

**工具与专题关联查询**：

```sql
SELECT b.type_name,
       b.type_code,
       b.pk_ability_type,
       a.ability_name,
       a.ability_code,
       string_agg(DISTINCT ''::text || a.pk_ability_basic::text, ','::text) AS pk_ability_basic
FROM rpa_ability_basic a
LEFT JOIN rpa_ability_type b ON a.pk_ability_type = b.pk_ability_type
GROUP BY a.ability_name, a.ability_code, b.type_name, b.type_code, b.pk_ability_type;
```

### 6.2 模板相关视图（1个）

| 视图名 | 说明 |
|--------|------|
| `view_ability_template` | 能力模板视图 |

### 6.3 资产相关表（7张）

| 表名 | 说明 |
|------|------|
| `g_asset` | 资产表 |
| `g_asset_type` | 资产类型表 |
| `g_asset_group` | 资产组表（业务系统） |
| `g_asset_assetgroup_rel` | 资产与资产组关联表（group_type=2 表示业务系统） |
| `g_asset_extend_prop` | 资产扩展属性表（用于查询是否上报，prop_code='REPORT_STATUS'） |
| `bd_checkitem` | 检查项表（基线 property=0，弱口令 property=9） |
| `bd_checkitem_related` | 检查项关联表（关联资产类型 pk_so） |

### 6.4 相关表结构SQL

**工具与专题关联查询**：

```sql

 SELECT b.type_name,
    b.type_code,
    b.pk_ability_type,
    a.ability_name,
    a.ability_code,
    string_agg(DISTINCT ''::text || a.pk_ability_basic::text, ','::text) AS pk_ability_basic
   FROM rpa_ability_basic a
     LEFT JOIN rpa_ability_type b ON a.pk_ability_type = b.pk_ability_type
  GROUP BY a.ability_name, a.ability_code, b.type_name, b.type_code, b.pk_ability_type;
```

**模板视图完整查询（view_ability_template）**：

```sql
SELECT all_template.foreign_template_id,
    all_template.foreign_template_name,
    all_template.pk_ability_ga_re,
    all_template.pk_ability_ga_ver,
    all_template.pk_ability_basic,
    all_template.ability_name,
    all_template.ability_code,
    all_template.type_code,
    all_template.source_form,
    all_template.is_default
   FROM ( SELECT tem.foreign_template_id,
            tem.foreign_template_name,
            tem.pk_ability_ga_re,
            agr.pk_ability_ga_ver,
            agr.pk_ability_basic,
            rab.ability_name,
            rab.ability_code,
            rat.type_code,
            0 AS source_form,
            '-1'::integer AS is_default
           FROM sc_template tem
             LEFT JOIN rpa_ability_ga_re agr ON agr.pk_ability_ga_re = tem.pk_ability_ga_re
             LEFT JOIN rpa_ability_basic rab ON agr.pk_ability_basic = rab.pk_ability_basic
             LEFT JOIN rpa_ability_type rat ON rab.pk_ability_type = rat.pk_ability_type
          GROUP BY tem.foreign_template_id, tem.foreign_template_name, tem.pk_ability_ga_re, agr.pk_ability_ga_ver, agr.pk_ability_basic, rab.ability_name, rab.ability_code, rat.type_code
        UNION ALL
         SELECT a.pk_task_template AS foreign_template_id,
            a.task_template_name AS foreign_template_name,
            b.pk_ability_ga_re,
            b.pk_ability_ga_ver,
            b.pk_ability_basic,
            b.ability_name,
            b.ability_code,
            b.type_code,
                CASE
                    WHEN a.is_internal = '1'::smallint THEN 0
                    ELSE 1
                END AS source_form,
                CASE
                    WHEN a.is_default = '1'::smallint THEN 1
                    ELSE 0
                END AS is_default
           FROM rs_task_template a
             JOIN ( SELECT 0 AS is_delete,
                    b_1.pk_ability_ga_re,
                    b_1.pk_ability_ga_ver,
                    b_1.pk_ability_basic,
                    a_1.ability_name,
                    a_1.ability_code,
                    e.type_code
                   FROM rpa_ability_basic a_1
                     JOIN rpa_ability_ga_re b_1 ON a_1.pk_ability_basic = b_1.pk_ability_basic
                     JOIN rpa_ability_ga_ver c ON c.pk_ability_ga_ver = b_1.pk_ability_ga_ver
                     JOIN rpa_ability_type e ON e.pk_ability_type = a_1.pk_ability_type
                  WHERE a_1.ability_code::text = 'ULTRA_ASSET_FOUND'::text AND c.ver_code::text = 'V1.0'::text) b ON a.is_delete = b.is_delete
          WHERE a.is_delete = 0 AND a.is_disabled = 0
        UNION ALL
         SELECT a.pk_collect_template AS foreign_template_id,
            a.template_name AS foreign_template_name,
            b.pk_ability_ga_re,
            b.pk_ability_ga_ver,
            b.pk_ability_basic,
            b.ability_name,
            b.ability_code,
            b.type_code,
                CASE
                    WHEN a.template_type = '0'::smallint THEN 0
                    ELSE 1
                END AS source_form,
                CASE
                    WHEN a.is_default = '1'::smallint THEN 1
                    ELSE 0
                END AS is_default
           FROM ac_collect_template a
             JOIN ( SELECT 0 AS is_delete,
                    b_1.pk_ability_ga_re,
                    b_1.pk_ability_ga_ver,
                    b_1.pk_ability_basic,
                    a_1.ability_name,
                    a_1.ability_code,
                    e.type_code
                   FROM rpa_ability_basic a_1
                     JOIN rpa_ability_ga_re b_1 ON a_1.pk_ability_basic = b_1.pk_ability_basic
                     JOIN rpa_ability_ga_ver c ON c.pk_ability_ga_ver = b_1.pk_ability_ga_ver
                     JOIN rpa_ability_type e ON e.pk_ability_type = a_1.pk_ability_type
                  WHERE a_1.ability_code::text = 'ULTRA_ASSET_COLLECT'::text AND c.ver_code::text = 'V1.0'::text) b ON a.is_delete = b.is_delete
          WHERE a.is_delete = 0 AND a.is_use = 1
        UNION ALL
         SELECT a.pk_handbook AS foreign_template_id,
            a.name AS foreign_template_name,
            b.pk_ability_ga_re,
            b.pk_ability_ga_ver,
            b.pk_ability_basic,
            b.ability_name,
            b.ability_code,
            b.type_code,
                CASE
                    WHEN a.source = '0'::smallint THEN 0
                    ELSE 1
                END AS source_form,
                CASE
                    WHEN a.is_default = '1'::smallint THEN 1
                    ELSE 0
                END AS is_default
           FROM bd_handbook a
             JOIN ( SELECT 0 AS is_delete,
                    b_1.pk_ability_ga_re,
                    b_1.pk_ability_ga_ver,
                    b_1.pk_ability_basic,
                    a_1.ability_name,
                    a_1.ability_code,
                    e.type_code
                   FROM rpa_ability_basic a_1
                     JOIN rpa_ability_ga_re b_1 ON a_1.pk_ability_basic = b_1.pk_ability_basic
                     JOIN rpa_ability_ga_ver c ON c.pk_ability_ga_ver = b_1.pk_ability_ga_ver
                     JOIN rpa_ability_type e ON e.pk_ability_type = a_1.pk_ability_type
                  WHERE a_1.ability_code::text = 'ULTRA_CONFIG_CHECK'::text AND c.ver_code::text = 'V1.0'::text) b ON a.is_delete = b.is_delete
          WHERE a.is_delete = 0 AND a.is_use = 1
        UNION ALL
         SELECT a.pk_handbook AS foreign_template_id,
            a.name AS foreign_template_name,
            b.pk_ability_ga_re,
            b.pk_ability_ga_ver,
            b.pk_ability_basic,
            b.ability_name,
            b.ability_code,
            b.type_code,
                CASE
                    WHEN a.source = '0'::smallint THEN 0
                    ELSE 1
                END AS source_form,
                CASE
                    WHEN a.is_default = '1'::smallint THEN 1
                    ELSE 0
                END AS is_default
           FROM bd_handbook a
             JOIN ( SELECT 0 AS is_delete,
                    b_1.pk_ability_ga_re,
                    b_1.pk_ability_ga_ver,
                    b_1.pk_ability_basic,
                    a_1.ability_name,
                    a_1.ability_code,
                    e.type_code
                   FROM rpa_ability_basic a_1
                     JOIN rpa_ability_ga_re b_1 ON a_1.pk_ability_basic = b_1.pk_ability_basic
                     JOIN rpa_ability_ga_ver c ON c.pk_ability_ga_ver = b_1.pk_ability_ga_ver
                     JOIN rpa_ability_type e ON e.pk_ability_type = a_1.pk_ability_type
                  WHERE a_1.ability_code::text = 'ULTRA_CLOUD_NATIVE'::text AND c.ver_code::text = 'V1.0'::text) b ON a.is_delete = b.is_delete
          WHERE a.is_delete = 0 AND a.is_use = 1 AND a.name::text ~~ '云原生_%'::text
        UNION ALL
         SELECT a.pk_handbook AS foreign_template_id,
            a.name AS foreign_template_name,
            b.pk_ability_ga_re,
            b.pk_ability_ga_ver,
            b.pk_ability_basic,
            b.ability_name,
            b.ability_code,
            b.type_code,
                CASE
                    WHEN a.source = '0'::smallint THEN 0
                    ELSE 1
                END AS source_form,
                CASE
                    WHEN a.is_default = '1'::smallint THEN 1
                    ELSE 0
                END AS is_default
           FROM bd_handbook a
             JOIN ( SELECT 0 AS is_delete,
                    b_1.pk_ability_ga_re,
                    b_1.pk_ability_ga_ver,
                    b_1.pk_ability_basic,
                    a_1.ability_name,
                    a_1.ability_code,
                    e.type_code
                   FROM rpa_ability_basic a_1
                     JOIN rpa_ability_ga_re b_1 ON a_1.pk_ability_basic = b_1.pk_ability_basic
                     JOIN rpa_ability_ga_ver c ON c.pk_ability_ga_ver = b_1.pk_ability_ga_ver
                     JOIN rpa_ability_type e ON e.pk_ability_type = a_1.pk_ability_type
                  WHERE a_1.ability_code::text = 'ULTRA_CONFIG_OFFLINE_CHECK'::text AND c.ver_code::text = 'V1.0'::text) b ON a.is_delete = b.is_delete
          WHERE a.is_delete = 0 AND a.is_use = 1
        UNION ALL
         SELECT a.pk_rule_template AS foreign_template_id,
            a.template_name AS foreign_template_name,
            b.pk_ability_ga_re,
            b.pk_ability_ga_ver,
            b.pk_ability_basic,
            b.ability_name,
            b.ability_code,
            b.type_code,
                CASE
                    WHEN a.template_type = 0::numeric THEN 0
                    ELSE 1
                END AS source_form,
                CASE
                    WHEN a.is_default = '1'::numeric THEN 1
                    ELSE 0
                END AS is_default
           FROM f_policy_rule_template a
             JOIN ( SELECT 0 AS is_delete,
                    b_1.pk_ability_ga_re,
                    b_1.pk_ability_ga_ver,
                    b_1.pk_ability_basic,
                    a_1.ability_name,
                    a_1.ability_code,
                    e.type_code
                   FROM rpa_ability_basic a_1
                     JOIN rpa_ability_ga_re b_1 ON a_1.pk_ability_basic = b_1.pk_ability_basic
                     JOIN rpa_ability_ga_ver c ON c.pk_ability_ga_ver = b_1.pk_ability_ga_ver
                     JOIN rpa_ability_type e ON e.pk_ability_type = a_1.pk_ability_type
                  WHERE a_1.ability_code::text = 'ULTRA_FIREWALL_CHECK'::text AND c.ver_code::text = 'V1.0'::text) b ON a.is_deleted = b.is_delete::numeric
          WHERE a.is_deleted = 0::numeric AND a.is_enable = 1::numeric
        UNION ALL
         SELECT a.pk_rule_template AS foreign_template_id,
            a.template_name AS foreign_template_name,
            b.pk_ability_ga_re,
            b.pk_ability_ga_ver,
            b.pk_ability_basic,
            b.ability_name,
            b.ability_code,
            b.type_code,
                CASE
                    WHEN a.template_type = 0::numeric THEN 0
                    ELSE 1
                END AS source_form,
                CASE
                    WHEN a.is_default = '1'::numeric THEN 1
                    ELSE 0
                END AS is_default
           FROM f_policy_rule_template a
             JOIN ( SELECT 0 AS is_delete,
                    b_1.pk_ability_ga_re,
                    b_1.pk_ability_ga_ver,
                    b_1.pk_ability_basic,
                    a_1.ability_name,
                    a_1.ability_code,
                    e.type_code
                   FROM rpa_ability_basic a_1
                     JOIN rpa_ability_ga_re b_1 ON a_1.pk_ability_basic = b_1.pk_ability_basic
                     JOIN rpa_ability_ga_ver c ON c.pk_ability_ga_ver = b_1.pk_ability_ga_ver
                     JOIN rpa_ability_type e ON e.pk_ability_type = a_1.pk_ability_type
                  WHERE a_1.ability_code::text = 'ULTRA_FIREWALL_OFFLINE_CHECK'::text AND c.ver_code::text = 'V1.0'::text) b ON a.is_deleted = b.is_delete::numeric
          WHERE a.is_deleted = 0::numeric AND a.is_enable = 1::numeric
        UNION ALL
         SELECT a.pk_weak_template AS foreign_template_id,
            a.template_name AS foreign_template_name,
            b.pk_ability_ga_re,
            b.pk_ability_ga_ver,
            b.pk_ability_basic,
            b.ability_name,
            b.ability_code,
            b.type_code,
                CASE
                    WHEN a.template_type = 0 THEN 0
                    ELSE 1
                END AS source_form,
                CASE
                    WHEN a.is_default = 1 THEN 1
                    ELSE 0
                END AS is_default
           FROM weak_template a
             JOIN ( SELECT 0 AS is_delete,
                    b_1.pk_ability_ga_re,
                    b_1.pk_ability_ga_ver,
                    b_1.pk_ability_basic,
                    a_1.ability_name,
                    a_1.ability_code,
                    e.type_code
                   FROM rpa_ability_basic a_1
                     JOIN rpa_ability_ga_re b_1 ON a_1.pk_ability_basic = b_1.pk_ability_basic
                     JOIN rpa_ability_ga_ver c ON c.pk_ability_ga_ver = b_1.pk_ability_ga_ver
                     JOIN rpa_ability_type e ON e.pk_ability_type = a_1.pk_ability_type
                  WHERE a_1.ability_code::text = 'ULTRA_PWD_CHECK'::text AND c.ver_code::text = 'V1.0'::text) b ON a.is_delete = b.is_delete
          WHERE a.is_delete = 0 AND a.is_use = 1
        UNION ALL
         SELECT a.pk_weak_template AS foreign_template_id,
            a.template_name AS foreign_template_name,
            b.pk_ability_ga_re,
            b.pk_ability_ga_ver,
            b.pk_ability_basic,
            b.ability_name,
            b.ability_code,
            b.type_code,
                CASE
                    WHEN a.template_type = 0 THEN 0
                    ELSE 1
                END AS source_form,
                CASE
                    WHEN a.is_default = 1 THEN 1
                    ELSE 0
                END AS is_default
           FROM weak_template a
             JOIN ( SELECT 0 AS is_delete,
                    b_1.pk_ability_ga_re,
                    b_1.pk_ability_ga_ver,
                    b_1.pk_ability_basic,
                    a_1.ability_name,
                    a_1.ability_code,
                    e.type_code
                   FROM rpa_ability_basic a_1
                     JOIN rpa_ability_ga_re b_1 ON a_1.pk_ability_basic = b_1.pk_ability_basic
                     JOIN rpa_ability_ga_ver c ON c.pk_ability_ga_ver = b_1.pk_ability_ga_ver
                     JOIN rpa_ability_type e ON e.pk_ability_type = a_1.pk_ability_type
                  WHERE a_1.ability_code::text = 'ULTRA_PWD_OFFLINE_CHECK'::text AND c.ver_code::text = 'V1.0'::text) b ON a.is_delete = b.is_delete
          WHERE a.is_delete = 0 AND a.is_use = 1
        UNION ALL
         SELECT a.pk_ca_template AS foreign_template_id,
            a.template_name AS foreign_template_name,
            b.pk_ability_ga_re,
            b.pk_ability_ga_ver,
            b.pk_ability_basic,
            b.ability_name,
            b.ability_code,
            b.type_code,
                CASE
                    WHEN a.template_type = '0'::smallint THEN 0
                    ELSE 1
                END AS source_form,
                CASE
                    WHEN a.is_default = '1'::smallint THEN 1
                    ELSE 0
                END AS is_default
           FROM ca_template a
             JOIN ( SELECT 0 AS is_delete,
                    b_1.pk_ability_ga_re,
                    b_1.pk_ability_ga_ver,
                    b_1.pk_ability_basic,
                    a_1.ability_name,
                    a_1.ability_code,
                    e.type_code
                   FROM rpa_ability_basic a_1
                     JOIN rpa_ability_ga_re b_1 ON a_1.pk_ability_basic = b_1.pk_ability_basic
                     JOIN rpa_ability_ga_ver c ON c.pk_ability_ga_ver = b_1.pk_ability_ga_ver
                     JOIN rpa_ability_type e ON e.pk_ability_type = a_1.pk_ability_type
                  WHERE a_1.ability_code::text = 'ULTRA_ASSET_ANALY'::text AND c.ver_code::text = 'V1.0'::text) b ON a.is_delete = b.is_delete
          WHERE a.is_delete = 0 AND a.is_use = 1
        UNION ALL
         SELECT a.pk_poc_template AS foreign_template_id,
            a.template_name AS foreign_template_name,
            b.pk_ability_ga_re,
            b.pk_ability_ga_ver,
            b.pk_ability_basic,
            b.ability_name,
            b.ability_code,
            b.type_code,
                CASE
                    WHEN a.template_type = 0 THEN 0
                    ELSE 1
                END AS source_form,
                CASE
                    WHEN a.is_default = 1 THEN 1
                    ELSE 0
                END AS is_default
           FROM poc_template a
             JOIN ( SELECT 0 AS is_delete,
                    b_1.pk_ability_ga_re,
                    b_1.pk_ability_ga_ver,
                    b_1.pk_ability_basic,
                    a_1.ability_name,
                    a_1.ability_code,
                    e.type_code
                   FROM rpa_ability_basic a_1
                     JOIN rpa_ability_ga_re b_1 ON a_1.pk_ability_basic = b_1.pk_ability_basic
                     JOIN rpa_ability_ga_ver c ON c.pk_ability_ga_ver = b_1.pk_ability_ga_ver
                     JOIN rpa_ability_type e ON e.pk_ability_type = a_1.pk_ability_type
                  WHERE a_1.ability_code::text = 'ULTRA_POC_CHECK'::text AND c.ver_code::text = 'V1.0'::text) b ON a.is_delete = b.is_delete
          WHERE a.is_delete = 0 AND a.is_use = 1
        UNION ALL
         SELECT a.pk_sd_template AS foreign_template_id,
            a.template_name AS foreign_template_name,
            b.pk_ability_ga_re,
            b.pk_ability_ga_ver,
            b.pk_ability_basic,
            b.ability_name,
            b.ability_code,
            b.type_code,
                CASE
                    WHEN a.template_type = 0 THEN 0
                    ELSE 1
                END AS source_form,
                CASE
                    WHEN a.is_default = 1 THEN 1
                    ELSE 0
                END AS is_default
           FROM sd_template a
             JOIN ( SELECT 0 AS is_delete,
                    b_1.pk_ability_ga_re,
                    b_1.pk_ability_ga_ver,
                    b_1.pk_ability_basic,
                    a_1.ability_name,
                    a_1.ability_code,
                    e.type_code
                   FROM rpa_ability_basic a_1
                     JOIN rpa_ability_ga_re b_1 ON a_1.pk_ability_basic = b_1.pk_ability_basic
                     JOIN rpa_ability_ga_ver c ON c.pk_ability_ga_ver = b_1.pk_ability_ga_ver
                     JOIN rpa_ability_type e ON e.pk_ability_type = a_1.pk_ability_type
                  WHERE a_1.ability_code::text = 'ULTRA_EXPOSE_MAPPING'::text AND c.ver_code::text = 'V1.0'::text) b ON a.is_deleted = b.is_delete
          WHERE a.is_deleted = 0 AND a.is_use = 1) all_template
     LEFT JOIN ( SELECT sy.pk_template,
            count(1) AS total_num,
            sum(
                CASE
                    WHEN sy.sync_status = 2 THEN 1
                    ELSE 0
                END) AS sync_num
           FROM bms_template_tool_sync sy
             LEFT JOIN rpa_ability_access b ON sy.pk_ability_access = b.pk_ability_access
          WHERE b.ability_status <> 0 AND b.is_deleted = 0
          GROUP BY sy.pk_template) syn_template ON syn_template.pk_template = all_template.foreign_template_id::bpchar
  WHERE all_template.source_form = 0 OR all_template.source_form <> 0 AND syn_template.sync_num = syn_template.total_num;
```

### 6.5 资产相关表DDL

**资产类型表（g_asset_type）**：

```sql
DROP TABLE IF EXISTS g_asset_type;
CREATE TABLE g_asset_type (
  pk_asset_type char(32) COLLATE pg_catalog.default NOT NULL,
  type_name varchar(500) COLLATE pg_catalog.default NOT NULL,
  type_code varchar(100) COLLATE pg_catalog.default NOT NULL,
  type_prop numeric(2,0),
  full_name varchar(3000) COLLATE pg_catalog.default,
  type_desc varchar(3000) COLLATE pg_catalog.default,
  pk_type_parent varchar(32) COLLATE pg_catalog.default,
  seriescode varchar(3300) COLLATE pg_catalog.default,
  display_idx int4,
  is_view numeric(2,0),
  come_from numeric(2,0),
  is_deleted numeric(2,0),
  pk_creator char(32) COLLATE pg_catalog.default,
  create_time timestamptz(6),
  pk_mender char(32) COLLATE pg_catalog.default,
  mend_time_last timestamptz(6),
  is_enable numeric(2,0),
  reserved1 varchar(500) COLLATE pg_catalog.default,
  reserved2 varchar(500) COLLATE pg_catalog.default,
  foreign_id varchar(100) COLLATE pg_catalog.default,
  foreign_md5 char(32) COLLATE pg_catalog.default,
  is_localize numeric(2,0),
  type_icon text COLLATE pg_catalog.default
)
;
ALTER TABLE g_asset_type OWNER TO postgres;
COMMENT ON COLUMN g_asset_type.type_prop IS '资产类型的类型：0=物理设备，1=应用软件';
COMMENT ON COLUMN g_asset_type.pk_type_parent IS '如果是根节点，值为：-1';
COMMENT ON COLUMN g_asset_type.is_view IS '0-不显示、1-显示';
COMMENT ON COLUMN g_asset_type.come_from IS '0-系统内置，1-自定义';
COMMENT ON COLUMN g_asset_type.is_deleted IS '0-未删除、1-已删除';
COMMENT ON TABLE g_asset_type IS '新增于：2022-04-06';


ALTER TABLE g_asset_type ADD CONSTRAINT g_asset_type_pkey PRIMARY KEY (pk_asset_type);
```

**资产扩展属性表（g_asset_extend_prop）**：

```sql
DROP TABLE IF EXISTS g_asset_extend_prop;
CREATE TABLE g_asset_extend_prop (
  pk_asset_extend_prop char(32) COLLATE pg_catalog.default NOT NULL,
  pk_asset char(32) COLLATE pg_catalog.default NOT NULL,
  prop_code varchar(100) COLLATE pg_catalog.default NOT NULL,
  prop_value varchar(3000) COLLATE pg_catalog.default
)
;
ALTER TABLE g_asset_extend_prop OWNER TO postgres;


CREATE INDEX idx_asset_extend_a ON g_asset_extend_prop USING btree (
  pk_asset COLLATE pg_catalog.default pg_catalog.bpchar_ops ASC NULLS LAST
);
CREATE INDEX idx_asset_extend_p ON g_asset_extend_prop USING btree (
  prop_code COLLATE pg_catalog.default pg_catalog.text_ops ASC NULLS LAST
);
CREATE INDEX idx_asset_extend_prop_asset_code ON g_asset_extend_prop USING btree (
  pk_asset COLLATE pg_catalog.default pg_catalog.bpchar_ops ASC NULLS LAST,
  prop_code COLLATE pg_catalog.default pg_catalog.text_ops ASC NULLS LAST
);


ALTER TABLE g_asset_extend_prop ADD CONSTRAINT pk_g_asset_extend_prop PRIMARY KEY (pk_asset_extend_prop);
```

**资产组表（g_asset_group）**：

```sql
DROP TABLE IF EXISTS g_asset_group;
CREATE TABLE g_asset_group (
  pk_group char(32) COLLATE pg_catalog.default NOT NULL,
  group_type numeric(2,0) NOT NULL,
  group_name varchar(500) COLLATE pg_catalog.default NOT NULL,
  group_code varchar(100) COLLATE pg_catalog.default,
  group_desc varchar(3000) COLLATE pg_catalog.default,
  full_name varchar(3000) COLLATE pg_catalog.default,
  parent_group_id char(32) COLLATE pg_catalog.default,
  series_code varchar(3300) COLLATE pg_catalog.default,
  sequence_idx int4,
  region_id varchar(100) COLLATE pg_catalog.default,
  deleted_state numeric(2,0),
  create_user char(32) COLLATE pg_catalog.default,
  create_time timestamptz(6),
  modify_user char(32) COLLATE pg_catalog.default,
  modify_time timestamptz(6),
  come_from numeric(2,0),
  foreign_id varchar(100) COLLATE pg_catalog.default,
  foreign_md5 char(32) COLLATE pg_catalog.default,
  reserved1 int4,
  reserved2 int4,
  reserved3 varchar(500) COLLATE pg_catalog.default,
  reserved4 varchar(500) COLLATE pg_catalog.default,
  major_region varchar(100) COLLATE pg_catalog.default,
  importance varchar(100) COLLATE pg_catalog.default,
  short_name varchar(500) COLLATE pg_catalog.default,
  group_num_id int8
)
;
ALTER TABLE g_asset_group OWNER TO postgres;
COMMENT ON COLUMN g_asset_group.group_type IS '1-组织机构、2-资产组、3-安全域';
COMMENT ON COLUMN g_asset_group.deleted_state IS '0：未删除 1：已删除';


CREATE INDEX idx_asset_group_seriescode ON g_asset_group USING btree (
  series_code COLLATE pg_catalog.default pg_catalog.text_ops ASC NULLS LAST
);


ALTER TABLE g_asset_group ADD CONSTRAINT pk_g_asset_group PRIMARY KEY (pk_group);

```