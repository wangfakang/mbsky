``nginx惊群问题简单分析：
``

# 内容：

## 一：惊群含义
## 二：nginx如何解决惊群
## 三：何时释放该锁
## 四：相应代码解析


### 一.惊群含义: 
  昨天所说的惊群含义是指:   
 
  * 当一个请求来到的时候,多个work进程的epoll注册监听端口都发生了可读事件,此时多个work进程都会去执行accept系统调用但最
终只有一个 work进程accept执行成功,造成了没有必要的上下文切换(这里的accept操作可以更深层次的"唤醒",此处所说的"唤醒"不是
说开始work进程都挂起的,然后把其唤醒,其实work进程几乎一直都是工作的如:处理一些之前已经建立好的一些网络事件).
 
### 二.此时nginx解决惊群是这样的    

  * 增加了一个"accept_mutex"锁,当其当前work进程抢到锁了才会去把监听端口的文件描述符加载到自己的epoll中,其他没有抢到的w
ork进程会把之前所注册的相应端口的事件全部删除,此时会处理之前建立连接上的一些网络事件.即"在同一个时刻只会有一个work进程
会在epoll中注册监听端口的描述符,每一个work进程都共享该监听端口描述符".


### 三.还有一个问题就是    
 
 * 抢到的锁何时释放,nginx是这样做的:使用了两个事件队列(ngx_posted_accept_events:可以理解为存放新建连接的事件,然后
ngx_posted_events:存放处理建立连接的事件),其实释放锁的地方就是:把新来的连接取出已经完成了三次握手了(此时把该事件放到ng
x_posted_events)然后就会释放锁.      
 * 锁释放了此时在处理ngx_posted_events(已经建立好连接的相关网络事件)上的事件,这样就可以让下次新的连接可以得到更快的
处理,后面再来新的连接的时候继续按照前面的处理方式处理.      
 * 注意:nginx的监听是采用的epoll的LT模式,有这样一个场景,就是当在处理新建连接中此时如果有一个新的请求又来了,由于监听
用epoll默认的模式(EPOLLT),所以新的请求被阻塞到监听套接口上,此时等释放了锁后其它进程取到了锁后会处理监听套接口上的连接.



### 四.代码解析
 
 ```  
    void
    ngx_process_events_and_timers(ngx_cycle_t *cycle)
    {
    ngx_uint_t  flags;
    ngx_msec_t  timer, delta;

    if (ngx_timer_resolution) {
        timer = NGX_TIMER_INFINITE;
        flags = 0;

    } else {
        timer = ngx_event_find_timer();
        flags = NGX_UPDATE_TIME;

    #if (NGX_THREADS)

        if (timer == NGX_TIMER_INFINITE || timer > 500) {
            timer = 500;
        }

    #endif
    }

    if (ngx_use_accept_mutex) {
        //表示当前进程的负载超过了最大连接数的7/8
        if (ngx_accept_disabled > 0) {
            ngx_accept_disabled--;

        } else {
            if (ngx_trylock_accept_mutex(cycle) == NGX_ERROR) {
                return;
            }

            if (ngx_accept_mutex_held) {
                flags |= NGX_POST_EVENTS;

            } else {// 不让其平凡的获取到锁   
                if (timer == NGX_TIMER_INFINITE
                    || timer > ngx_accept_mutex_delay)
                {
                    timer = ngx_accept_mutex_delay;
                }
            }
        }
    }

    delta = ngx_current_msec;

    (void) ngx_process_events(cycle, timer, flags);

    delta = ngx_current_msec - delta;

    ngx_log_debug1(NGX_LOG_DEBUG_EVENT, cycle->log, 0,
                   "timer delta: %M", delta);
    //处理新来的事件建立连接
    if (ngx_posted_accept_events) {
        ngx_event_process_posted(cycle, &ngx_posted_accept_events);
    }
    //处理完了就把其锁给释放了,让其新的连接可以被其尽快处理
    if (ngx_accept_mutex_held) {
        ngx_shmtx_unlock(&ngx_accept_mutex);
    }

    if (delta) {
        ngx_event_expire_timers();
    }

    ngx_log_debug1(NGX_LOG_DEBUG_EVENT, cycle->log, 0,
                   "posted events %p", ngx_posted_events);
    //锁释放了才处理已经建立好的事件(已经完成了三次握手的事件)
    if (ngx_posted_events) {
        if (ngx_threaded) {
            ngx_wakeup_worker_thread(cycle);

        } else {
            ngx_event_process_posted(cycle, &ngx_posted_events);
        }
    }
    }
    
 ```    
    
 ```       
    ngx_int_t
    ngx_trylock_accept_mutex(ngx_cycle_t *cycle)
    {
    //获取锁操作
    if (ngx_shmtx_trylock(&ngx_accept_mutex)) {

        ngx_log_debug0(NGX_LOG_DEBUG_EVENT, cycle->log, 0,
                       "accept mutex locked");
       //获取到锁，但是标志位ngx_accept_mutex_held为1，表示当前进程已经获取到锁    
        if (ngx_accept_mutex_held
            && ngx_accept_events == 0
            && !(ngx_event_flags & NGX_USE_RTSIG_EVENT))
        {
            return NGX_OK;
        }
        //获取到锁,则将所有监听事件添加到当前的epoll等事件驱动模块中  
        if (ngx_enable_accept_events(cycle) == NGX_ERROR) {
            ngx_shmtx_unlock(&ngx_accept_mutex);
            return NGX_ERROR;
        }

        ngx_accept_events = 0;
        ngx_accept_mutex_held = 1;

        return NGX_OK;
    }

    ngx_log_debug1(NGX_LOG_DEBUG_EVENT, cycle->log, 0,
                   "accept mutex lock failed: %ui", ngx_accept_mutex_held);
    //表示此次没有获取到锁,但是上次获取到锁,此时会把当前work的epoll中删除监听的端口事件
    if (ngx_accept_mutex_held) {
        if (ngx_disable_accept_events(cycle) == NGX_ERROR) {
            return NGX_ERROR;
        }

        ngx_accept_mutex_held = 0;
    }

    return NGX_OK;
    }

  ```   

## 有问题反馈
在使用中有任何问题，欢迎反馈给我，可以用以下联系方式跟我交流

* 邮件(1031379296#qq.com, 把#换成@)
* QQ: 1031379296
* weibo: [@王发康](http://weibo.com/u/2786211992/home)


## 关于作者

### Linux\nginx\golang\c\c++爱好者
### 欢迎一起交流  一起学习
