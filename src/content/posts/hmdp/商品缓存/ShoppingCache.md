---
title: 短信登陆功能
published: 2026-06-19
# updated: 2024-11-29
description: 利用缓存存储商品数据
image: ''
tags: [cache,hmdp]
category: hmdp
draft: false 
---
# 商品查询缓存
## 什么是缓存（cache）
缓存就是数据交换的缓冲区，是存储数据的临时地方，一般读写性较高。

### 缓存的优缺点
—优点：降低后端负载，提高读写效率，降低响应时间
—缺点：数据不一致，代码维护成本，运营成本

## 添加商品缓存原理
![缓存的原理](1.png)
>[!TIP]
>相比较于之前的逻辑，这里先会查询redis，没有在查数据库，再加入redis中

### 代码实现
~~~java
//controller层：
 @GetMapping("/{id}")
    public Result queryShopById(@PathVariable("id") Long id) {
        return shopService.queryById(id);
    }//这里新定义一个方法，先查redis
//Service层：
@Service
public class ShopServiceImpl extends ServiceImpl<ShopMapper, Shop> implements IShopService {
    @Resource
    private StringRedisTemplate stringRedisTemplate;

    @Override
    public Object quertById(Long id) {
        String key = CACHE_SHOP_KEY + id;
        //从redis里查询
        String shopJson = stringRedisTemplate.opsForValue().get(key);

        //查到了，转成bean返回
        if(!StrUtil.isNotBlank(shopJson)){
            Shop shop = JSONUtil.toBean(shopJson,Shop.class);
            return Result.ok(shop);
        }
        //不存在，根据id查询数据库
        Shop shop = getById(id);

        //不存在数据库返回错误
        if(shop == null){
            return Result.fail("店铺不存在！");
        }

        //存在，写入redis
        stringRedisTemplate.opsForValue().set(key,JSONUtil.toJsonStr(shop));
        return Result.ok(shop);
    }
}
~~~
>[!NOTE]
>这里用了很多Hutool里的方法:<br>
1.StrUtil.isNotBlank():传入的是字符串，判断是否为空，即便是"  "也算作空，不为空返回true;<br>
>2.JSONUtil.toBean():作用是将字符串转化成bean对象，传入的参数，一个是转化的对象，另一个是要转化的对象类.<br>
>3.JSONUtil.toJsonStr():就是将一个对象转化为JSON字符串

## 商品分类查询缓存
思路一致，但是查询的结果是list集合，因为有很多分类，这里采用两种方式分别是string,hash,list的redis查询
### String形式存入redis
~~~java
//controller:
 @GetMapping("list")
    public Result queryTypeList() {
        return typeService.queryList();
    }
//Service层:
 Result queryList();


@Service
public class ShopTypeServiceImpl extends ServiceImpl<ShopTypeMapper, ShopType> implements IShopTypeService {

    @Resource
    private StringRedisTemplate stringRedisTemplate;
    @Override
    public Result queryList() {
        String shopsort=stringRedisTemplate.opsForValue().get(SHOP_SORT_KEY);

        if(StrUtil.isNotBlank(shopsort)){
            List<ShopType>list= JSONUtil.toList(shopsort,ShopType.class);
            return Result.ok(list);
        }

        //redis中没有，查询数据库,这里的意思是查询然后按照sort进行排序升序 ASC，最后返回list的类型
        List<ShopType> typeList = query().orderByAsc("sort").list();
        //数据库中没有，报错
        if (CollectionUtil.isEmpty(typeList)) {
            return Result.fail("列表信息不存在");
        }
       stringRedisTemplate.opsForValue().set(SHOP_SORT_KEY,JSONUtil.toJsonStr(typeList)); 
       return Result.ok(typeList);

    }
}

//存list
 @Override
    public Result queryList() {
        //使用list取 [0 -1] 代表全部
        //从redis中查询所有分类
        List<String> shopTypeList = redisTemplate.opsForList().range(SHOP_LIST_KEY, 0, -1);
        if (CollectionUtil.isNotEmpty(shopTypeList)) {
            //shopTypeList.get(0) 其实是获取了整个List集合里的元素,0是第0个key
            //查询到，直接返回
            List<ShopType> types = JSONUtil.toList(shopTypeList.get(0), ShopType.class);
            return Result.ok(types);
        }
 
        //redis中没有，查询数据库
        List<ShopType> typeList = query().orderByAsc("sort").list();
        //数据库中没有，报错
        if (CollectionUtil.isEmpty(typeList)) {
            return Result.fail("列表信息不存在");
        }
 
        //list 存
        //数据库中有，存到redis，注意存的位置和上面取的位置一致。
        String jsonStr = JSONUtil.toJsonStr(typeList);
        redisTemplate.opsForList().leftPushAll(SHOP_LIST_KEY, jsonStr);
        return Result.ok(typeList);
 
    }
~~~