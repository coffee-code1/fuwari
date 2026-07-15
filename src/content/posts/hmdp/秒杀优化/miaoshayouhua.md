---
title: 点评项目之-五.秒杀优化
published: 2026-07-14
updated: 2026-07-14
description: 解决之前的优惠券秒杀业务时间太长问题
image: ''
tags: [lua,点评项目,Redisson,分布式锁]
category: Redis
draft: false 
---
- [一、原先业务的缺点](#一原先业务的缺点)
- [二、解决方法](#二解决方法)
  - [2.1 如何实现秒杀资格业务](#21-如何实现秒杀资格业务)
  - [2.2 代码实操](#22-代码实操)

# 一、原先业务的缺点
是单线程顺序执行的，只有**前面的业务执行完，后面才会执行**，而且要**查库**，这都会损耗业务时长，当并发时**效率更低**

# 二、解决方法
1.采取**异步，**在判断优惠券是否有库存时，在开另一个线程执行减库存跟下单操作
2.在判断是否有秒杀资格可以把库存数据，订单信息放在**redis**中，减少查询速率，如果有资格将查询的**订单id，用户id等信息放入**阻塞队列，以便于后续的另一个线程的处理
3.这样就分成了两块，一个是秒杀资格的判断，并记录优惠券id，用户id，订单id，二个就是将以上的信息存入阻塞队列。

## 2.1 如何实现秒杀资格业务
1.采取set集合存储订单,订单id作为key，用户id作为值，这样不重复，也可以存储多个用户，实现一人一单
2.采取string存储库存
>[!TIP]
全部是lua脚本实现的，这样确保了原子性

![实现了思路](1.png)

## 2.2 代码实操
1.添加优惠券时存入redis
~~~java
@Service
public class VoucherServiceImpl extends ServiceImpl<VoucherMapper, Voucher> implements IVoucherService {


    @Resource
    private StringRedisTemplate stringRedisTemplate;

    @Override
    public Result queryVoucherOfShop(Long shopId) {
        // 查询优惠券信息
        List<Voucher> vouchers = getBaseMapper().queryVoucherOfShop(shopId);
        // 返回结果
        return Result.ok(vouchers);
    }

    @Override
    @Transactional
    public void addSeckillVoucher(Voucher voucher) {
        // 保存优惠券
        save(voucher);
        // 保存秒杀信息
        SeckillVoucher seckillVoucher = new SeckillVoucher();
        seckillVoucher.setVoucherId(voucher.getId());
        seckillVoucher.setStock(voucher.getStock());
        seckillVoucher.setBeginTime(voucher.getBeginTime());
        seckillVoucher.setEndTime(voucher.getEndTime());
//        seckillVoucherService.save(seckillVoucher);
        //这里采用redis存储而不是数据库
        stringRedisTemplate.opsForValue().set(SECKILL_STOCK_KEY+seckillVoucher.getVoucherId(),seckillVoucher.getStock().toString());
    }

}
~~~
2.编写lua实现上述逻辑
~~~lua

---秒杀券id
local voucherId = ARGV[1]

---用户id
local userId = ARGV[2]

---库存key
local stockkey = "seckill:stock:" .. voucherId

---订单key
local orderkey = "seckill:order" .. voucherId

---判断库存是否充足
if(tonumber(redis.call('get',stockkey))<=0)then
    return 1
end

---判断是否重复下单
if(redis.call('sismember',orderkey,userId) == 1)then
    return 2
end

---满足条件，库存减一
redis.call('incrby',stockkey,-1)
---满足条件，保存用户id
redis.call('sadd',orderkey,userId)

return 0
~~~
3.下单秒杀券业务
~~~java
@Service
@Slf4j
public class VoucherOrderServiceImpl extends ServiceImpl<VoucherOrderMapper, VoucherOrder> implements IVoucherOrderService {

    @Resource
    private ISeckillVoucherService seckillVoucherService;

    @Resource
    private RedisIDWorker redisIDWorker;

    @Resource
    private StringRedisTemplate stringRedisTemplate;

    @Resource
    private RedissonClient redissonClient;

    private static final DefaultRedisScript<Long> SECKILL_SCRIPT;

    static {
        SECKILL_SCRIPT = new DefaultRedisScript<>();
        SECKILL_SCRIPT.setLocation(new ClassPathResource("Seckill.lua"));
        SECKILL_SCRIPT.setResultType(Long.class);
    }

    /**
     * Save successful seckill requests and let a background thread create orders.
     */
    private final BlockingQueue<VoucherOrder> orderTasks = new ArrayBlockingQueue<>(1024 * 1024);

    /**
     * Process queued order tasks in a single thread to reduce write contention.
     */
    private static final ExecutorService SECKILL_ORDER_EXECUTOR = Executors.newSingleThreadExecutor();

    private IVoucherOrderService proxy;

    //保证在类加载完后，执行下面代码进行初始化，生成线程池
    @PostConstruct
    private void init() {
        SECKILL_ORDER_EXECUTOR.submit(new VoucherOrderHandler());
    }

    //在类加载完毕后执行init后，就会一直调用死循环不断地从阻塞队列中获取信息，并调用创建订单的方法
    private class VoucherOrderHandler implements Runnable {
        @Override
        public void run() {
            while (true) {
                try {
                    VoucherOrder voucherOrder = orderTasks.take();
                    handleVoucherOrder(voucherOrder);
                } catch (Exception e) {
                    log.error("handle voucher order failed", e);
                }
            }
        }
    }

    private void handleVoucherOrder(VoucherOrder voucherOrder) {
        Long userId = voucherOrder.getUserId();
        RLock lock = redissonClient.getLock("Order:" + userId);
        boolean isLock = lock.tryLock();
        if (!isLock) {
            log.error("不能重复下单");
            return;
        }
        try {
            proxy.createVoucherOrder(voucherOrder);
        } finally {
            lock.unlock();
        }
    }

    @Override
    public Result seckillVoucher(Long id) {
        Long userId = UserHolder.getUser().getId();
        Long result = stringRedisTemplate.execute(
                SECKILL_SCRIPT,
                Collections.emptyList(),
                id.toString(), userId.toString()
        );
        int r = result.intValue();
        if (r != 0) {
            return Result.fail(r == 1 ? "库存不足" : "不能重复下单");
        }

        // Build the order first, then let the background thread persist it.
        //创建订单
        long orderId = redisIDWorker.nextId("order");
        VoucherOrder voucherOrder = new VoucherOrder();
        voucherOrder.setId(orderId);
        voucherOrder.setUserId(userId);
        voucherOrder.setVoucherId(id);

        //给代理对象赋值
        proxy = (IVoucherOrderService) AopContext.currentProxy();
        orderTasks.add(voucherOrder);
        return Result.ok(orderId);
    }

    //老版本的保存订单到库
    @Transactional
    @Override
    public Result createVoucherOrder(Long id) {
        boolean success = seckillVoucherService
                .update().setSql("stock = stock - 1")
                .eq("voucher_id", id)
                .gt("stock", 0)
                .update();
        if (!success) {
            return Result.fail("修改失败");
        }

        VoucherOrder voucherOrder = new VoucherOrder();
        Long userId = UserHolder.getUser().getId();
        voucherOrder.setUserId(userId);
        voucherOrder.setVoucherId(id);
        Long orderId = redisIDWorker.nextId("order");
        voucherOrder.setId(orderId);
        save(voucherOrder);
        return Result.ok(orderId);
    }

    //当下适配的做法
    @Transactional
    @Override
    public void createVoucherOrder(VoucherOrder voucherOrder) {
        boolean success = seckillVoucherService
                .update().setSql("stock = stock - 1")
                .eq("voucher_id", voucherOrder.getVoucherId())
                .gt("stock", 0)
                .update();
        if (!success) {
            log.error("stock not enough, create order failed, voucherId={}", voucherOrder.getVoucherId());
            return;
        }
        save(voucherOrder);
    }
}
~~~
>[!NOTE]
代码执行顺序是：当前类加载完毕后会由于**PostConstruct**加载内容，线程池就会创建一个子线程，子线程就会进入while循环，不断获取阻塞队列里的信息，有就调用创建订单的方法，没有就会等待。并且这是异步，同时主线程会继续执行下面的代码，判断新的请求是否具有下单资格。

>[!WARNING]
这里的分布式锁在这里**单机器下是多余的**，因为不是集群，只有一台服务，**lua原子性**已经解决不会出现超卖问题了