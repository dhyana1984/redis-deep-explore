
# 设计思想
漏斗限流是最常用的限流方式之一，其来源灵感于漏斗。漏斗的容量是有限的，如果将溜嘴堵住，然后一直往里面灌水，它就会变满，直至再也装不进去。如果将漏嘴放开，水就会往下流，流走一部分后，又可以继续往里面灌水。如果漏嘴流水的速率大于灌水的速率，那么漏斗就永远都装不满。如果漏嘴流水速率小于灌水速率，那么一旦漏斗满了，灌水就需要暂停并等待漏斗腾出一部分空间。

所以，**漏斗的剩余空间代表当前行为可以持续进行的数量，漏嘴的流水速率代表着系统允许该行为的最大频率。**
<!-- more -->
# 单机漏斗实现
Funnel 对象的 nakeWater 方法是漏斗算法的核心，每次灌水之前都会被调用以触发漏水，给漏斗腾出空间来。能腾出多少空间取决于过去了多久以及流水的速率。Funnel 对象占据的空间不再和行为的频率成正比，它的空间占用是一个常量。

```java
public class FunnelRateLimiter {
    private Map<String, Funnel> funnelMap = new HashMap<>();

    public static void main(String[] args) {
        FunnelRateLimiter limiter = new FunnelRateLimiter();
        for (int i = 0; i < 20; i++) {
            // 流水速率 quota/s 设置为 0.1，一分钟最多 6 个。
            System.out.println(limiter.isActionAllowed("Harry", "reply", 6, 0.1f));
        }
    }

    public boolean isActionAllowed(String userId, String actionKey, int capacity, float leakingRate) {
        String key = String.format("%s:%s", actionKey, userId);
        Funnel funnel = funnelMap.get(key);
        if (null == funnel) {
            funnel = new Funnel(capacity, leakingRate);
            funnelMap.put(key, funnel);
        }
        return funnel.watering(1);
    }


    /**
     * 漏斗容器
     */
    static class Funnel {
        // 漏斗容量
        int capacity;
        // 漏嘴流水速率 quota/s
        float leakingRate;
        // 漏斗剩余容量
        int leftQuota;
        // 上一次流水时间
        long leakingTs;

        public Funnel(int capacity, float leakingRate) {
            this.capacity = capacity;
            this.leftQuota = capacity;
            this.leakingRate = leakingRate;
            this.leakingTs = System.currentTimeMillis();
        }

        /**
         * 漏斗漏水，释放空间
         */
        void makeWater() {
            long nowTs = System.currentTimeMillis();
            // 距离上一次漏水的时间差（上一次漏了多长时间）
            long deltaTs = nowTs - leakingTs;
            // 到这次漏水时，上一次漏水一共流了多少水（腾出了多少空间）
            int deltaQuota = (int) (deltaTs * leakingRate);
            // 间隔时间太长，整数数字过大溢出
            if (deltaQuota < 0) {
                this.leftQuota = capacity;
                this.leakingTs = nowTs;
                return;
            }
            // 腾出的空间太小，最小单位为 1
            if (deltaQuota < 1) {
                return;
            }
            this.leftQuota += deltaQuota;
            this.leakingTs = nowTs;
            if (this.leftQuota > this.capacity) {
                this.leftQuota = this.capacity;
            }
        }

        /**
         * 往漏斗灌水
         *
         * @param quota 灌入容量
         * @return 是否灌入成功
         */
        boolean watering(int quota) {
            makeWater();
            if (this.leftQuota >= quota) {
                this.leftQuota -= quota;
                return true;
            }
            return false;
        }
    }
}
```

如果将以上单机版的改成分布式的，使用 redis 怎么实现？观察 Funnel 对象的四个属性，可以联想到 hash 结构。灌水的时候将 hash 结构的属性取出来进行逻辑计算后，再放回到 hash 中完成了一次行为频度的检测。但是这种方式有个问题，取值、内存计算、回填值不是一个原子操作，意味着会有数据不准确的问题。

怎么保证原子操作？由此联想到分布式锁，用了分布式锁就可能会有加锁失败，加锁失败的话选择重试或放弃。重试的话就造成性能下降，放弃的话用户的操作就失败了体验不好。好在 redis4.0 之后提供了限流模块解决这种问题。
M
# Redis-Cell
redis4.0 提供的限流模块，该模块只有一个指令 `cl.throttle`。

`cl.throttle <key> <max_burst> <count per period> <period> [<quantity>]`

【参数】
1. key: 键名
2. max_burst: 最大容量
3. count per period: 每个周期的数量，漏水速率。
4. period: 周期时长
5. quantity: 每次漏水数量，默认为 1（可选）

```sh
127.0.0.1:6379> cl.throttle reply:tom 5 5 60 1
1) (integer) 0  # 0 表示允许，1 表示拒绝
2) (integer) 6  # 总容量
3) (integer) 4  # 剩余容量
4) (integer) -1 # 如果被拒绝，需要多少时间后重试。
5) (integer) 21 # 多长时间，漏斗完全空出来。
```

以上命令的意思是初始容量为 5，60s 之内能够限制只能回复 5 次，单位是 1。一开始可以连续回复 5 次，后续才收到漏水速率的影响。怎么理解？

> 通俗来讲，每次调用 cl.throttle 指令都看作是向漏斗里面灌水，刚开始漏斗里面没有水，初始容量为5，我们可以连续灌入五次一个单位的水（灌水速率远大于漏水速率，所以可以立即灌满）。满了之后，60s 之内会漏出 5 个单位的水，所以我们在 60s 之内能够灌水 5 次。

练习：project/1-9
