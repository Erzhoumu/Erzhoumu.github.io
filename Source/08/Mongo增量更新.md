---
title: Mongo增量更新
date: 2017-08-14
tags: Mongo
---
##### 问题描述：
在开发过程中，我们发现对某个类，在前端进行部分字段的编辑，那么在后端进行存储时需要将其它未变更字段重新存储或者在前端添加hidden字段，如果类字段太多就是费时费力了，并且很容易留下坑，造成数据损失。
##### 需求：
只对变更的字段进行存储修改。
##### 实现：
``` bash
/**
 * Mongo的增量更新;目前支持基本类型;以及Array;Set和List,类请继承BaseObject
 * @Attention 只支持对原有数据的增加和修改,不支持删除;static 类型不更新; @JSONField(serialize=false)不更新
 * @Attention 类结构中的基础类型需使用包装类型（int->Integer）
 * @author  Nil
 * @date 2017-7-22
 *
 * As the last ship sailed towards the distant horizon
 *      I sat there watching on a rock
 *       My mind slowly drifting away
 *       Forming into my dream tale
 *                       -The Dawn
 */
@Service
public class MongoCommonService {

    @Autowired
    private MongoTemplate mongoTemplate;

    private static Logger logger = LoggerFactory.getLogger(MongoCommonService.class);

    private Update doWithFields(final Object srcObject, final Update update, final String parentFieldName) {

        if(srcObject == null) {
            return update;
        }
        Class srcClass = (Class)srcObject.getClass();

        ReflectionUtils.doWithFields(srcClass, new ReflectionUtils.FieldCallback() {
            @Override
            public void doWith(Field field) throws IllegalArgumentException, IllegalAccessException {
                doWithField(srcObject, field, update, parentFieldName);
            }
        });
        return update;
    }

    private Update doWithField (Object object, Field field, Update update, String parentFieldName) throws IllegalAccessException {

        boolean access = field.isAccessible();
        field.setAccessible(true);

        Object var = field.get(object);
        if(var == null) {
            return update;
        }

        //static 不更新
        if(Modifier.isStatic(field.getModifiers())){
            return update;
        }
        //JsonField.serialize = false不更新
        if(field.isAnnotationPresent(JSONField.class) && !field.getAnnotation(JSONField.class).serialize()){
            return update;
        }

        Class<?> type = field.getType();
        String fieldName = field.getName();
        String updateField = parentFieldName == null?fieldName:parentFieldName+"."+fieldName;
        if(type.isPrimitive() || type == String.class || type == Long.class || type == Integer.class || type == Date.class) {
            //简单类型直接赋值
            update.set(updateField, var);
        } else if(type.isArray()) {
            //数组只会更新和增加不会删除
            int length = Array.getLength(var);
            //对每个item进行处理
            for (int i = 0; i < length; i++) {
                Object item = Array.get(var, i);
                doWithFields(item, update, updateField + "." +i);
            }
        } else if(List.class.isAssignableFrom(type)){
            //对List操作
            String methodKey = "get" + fieldName.substring(0, 1).toUpperCase() + fieldName.substring(1);
            Method m;
            try {
                m = object.getClass().getMethod(methodKey);
                List list = (List) m.invoke(object);
                for(int i = 0; i < list.size(); i++) {
                    Object item = list.get(i);
                    doWithFields(item,update, updateField + "." +i);
                }
            } catch (InvocationTargetException | NoSuchMethodException e) {
                logger.error("[ 更新结构失败]:List={},msg={}", updateField, e.getMessage());
            }
        } else if(Set.class.isAssignableFrom(type)){
            //对Set操作
            String methodKey = "get" + fieldName.substring(0, 1).toUpperCase() + fieldName.substring(1);
            Method m;
            try {
                m = object.getClass().getMethod(methodKey);
                Set set = (Set) m.invoke(object);
                int index = 0;
                while(set.iterator().hasNext()){
                    Object item = set.iterator().next();
                    doWithFields(item,update, updateField + "." +index);
                    index ++;
                }
            } catch (InvocationTargetException | NoSuchMethodException e) {
                logger.error("[ 更新结构失败]:set={},msg={}", updateField, e.getMessage());
            }
        } else if (BaseObject.class.isAssignableFrom(type)) {
            //如果是类结构
            doWithFields(var, update, updateField);
        }
        field.setAccessible(access);
        return update;
    }

    /**
     * 只更新已经存在的
     * @param query
     * @param baseObj
     * @param collection
     */

    public void dirtyUpdateFirst(Query query,Object baseObj, String collection){
        Update update = new Update();
        update = doWithFields(baseObj, update, null);
        mongoTemplate.updateFirst(query,update, collection);
    }

    /**
     * 更新;不存在会新建
     * @param query
     * @param baseObj
     * @param collection
     */
    public void dirtyUpsert (Query query,Object baseObj, String collection){
        Update update = new Update();
        update = doWithFields(baseObj, update, null);
        mongoTemplate.upsert(query,update, collection);
    }
}
```
##### 参考：
http://www.jianshu.com/p/dd7b5a0e2f64
##### 技术栈：
+ 反射机制
+ Mongo更新