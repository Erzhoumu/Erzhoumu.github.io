---
title: import（批量导入）
date: 2017-08-14
tags: web, tools
---
+ 介绍：
类似于导出，再主线程内进行长时间的任务都是作死。
+ 需求：
    + 异步进行；
    + 字段统一配置；
+ 思路：
    + 统一配置管理：
        + 字段特征（Name， 导入规则(nullAble，字段类型, 格式)）；（CellDef<AssetVO>）
        + example: 
            + ``` java
        cells.add(new CellDef<>("参数组id", false, new CalCellFun<AssetVO>() {
    @Override
    public String calCell(String name, String value, boolean nullable, AssetVO vo) {
        StringBuilder msg = new StringBuilder();
        int groupId;
        try{
            groupId = Integer.valueOf(value);
        } catch (Exception e) {
            msg.append(name + "格式错误!\r\n");
            return msg.toString();
        }
        vo.setFinGroupId(groupId);
        return msg.toString();
    }
}));
```
    + 过程标识
        + redis保存一个状态值，完成时改为1；
    + 错误信息
        + redis进行保存，结束时可获取；
    + 流程
        + readExcel -> importToDB (BatchImportFun)
        + readExcel 
            + 抽象Cell（） ->cellsList