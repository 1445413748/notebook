### synchronized缺陷

+ 无法控制阻塞时长

  比如以下代码

  ```java
  public class SyncTest {
  
      public synchronized void syncMethod(){
          try{
              TimeUnit.HOURS.sleep(1);
          }catch (InterruptedException e){
              e.printStackTrace();
          }
      }
  
      public static void main(String[] args) throws InterruptedException{
          SyncTest syncTest = new SyncTest();
          Thread t1 = new Thread(syncTest::syncMethod, "t1");
          Thread t2 = new Thread(syncTest::syncMethod, "t2");
          t1.start();
          TimeUnit.MILLISECONDS.sleep(2);
          t2.start();
      }
  }
  ```

  t1 进入同步方法 syncMethod 后会休眠一个小时，当 t2 启动后，因为拿不到锁，所以进入阻塞状态。

  t2 什么时候执行取决于  t1 什么时候释放锁。

  如果现在要 t2 最多阻塞一分钟，不然就放弃，这在这里是没办法实现的。

+ 阻塞没办法被中断

  上面的代码修改一下 main 方法

  ```java
  public static void main(String[] args) throws InterruptedException{
      SyncTest syncTest = new SyncTest();
      Thread t1 = new Thread(syncTest::syncMethod, "t1");
      Thread t2 = new Thread(syncTest::syncMethod, "t2");
      t1.start();
      TimeUnit.MILLISECONDS.sleep(2);
      t2.start();
      TimeUnit.MILLISECONDS.sleep(2);
      t2.interrupt();
      System.out.println(t2.isInterrupted()); //ture
      System.out.println(t2.getState()); //BLOCKED
      }
  
  ```





Java SDK 并发包通过 Lock 和 Condition 两个接口来实现管程，其中 Lock 用于解决互斥问题，Condition 用于解决同步问题。

### Lock

