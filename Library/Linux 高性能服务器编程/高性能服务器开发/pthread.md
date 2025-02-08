`pthread_cond_wait(&worker->thread_poll->cond, &worker->thread_poll->mutex);` 是一个用于线程同步的函数。以下是具体解释：

- `pthread_cond_wait` 是一个阻塞函数，用于等待条件变量 `cond` 被触发。
- 在调用 `pthread_cond_wait` 时，当前线程会释放互斥锁 `mutex` 并进入挂起等待状态，直到条件变量 `cond` 被其他线程触发（通过 `pthread_cond_signal` 或 `pthread_cond_broadcast`）。
- 当条件变量 `cond` 被触发后，`pthread_cond_wait` 会重新锁定互斥锁 `mutex` 并返回，继续执行后续代码。

具体步骤如下：

1. 当前线程持有互斥锁 `mutex`。
2. 调用 `pthread_cond_wait`，当前线程释放互斥锁 `mutex` 并进入等待状态。
3. 其他线程可以在此期间修改共享资源，并通过 `pthread_cond_signal` 或 `pthread_cond_broadcast` 触发条件变量 `cond`。
4. 条件变量 `cond` 被触发后，`pthread_cond_wait` 重新锁定互斥锁 `mutex` 并返回，当前线程继续执行。

---

`pthread_cond_signal` 和 `pthread_cond_broadcast` 都是用于操作条件变量的函数，主要区别在于它们的通知方式：

1. **`pthread_cond_signal`**
    
    - **作用**：唤醒**一个**正在等待该条件变量的线程。
    - **用法**：当一个线程满足某个条件并希望通知其他线程时，调用 `pthread_cond_signal` 可以唤醒一个正在等待该条件的线程。如果有多个线程在等待条件变量，调用 `pthread_cond_signal` 只会唤醒其中一个线程。
    - **典型使用场景**：如果你知道只有一个线程能够继续执行，比如生产者-消费者模型中的生产者生产一个产品后只唤醒一个消费者线程。
    
    ```c
    int pthread_cond_signal(pthread_cond_t *cond);
    ```
    
2. **`pthread_cond_broadcast`**
    
    - **作用**：唤醒**所有**正在等待该条件变量的线程。
    - **用法**：当多个线程可能都需要被唤醒时，调用 `pthread_cond_broadcast` 会通知所有等待该条件变量的线程，使得它们都能够重新竞争锁资源并继续执行。
    - **典型使用场景**：如果多个线程都可能因为条件变化而需要继续执行，比如在生产者-消费者模型中，当多个消费者都在等待多个产品时，生产者可能希望唤醒所有消费者。
    
    ```c
    int pthread_cond_broadcast(pthread_cond_t *cond);
    ```
    

### 主要区别：

|特性|`pthread_cond_signal`|`pthread_cond_broadcast`|
|---|---|---|
|**唤醒的线程数**|唤醒一个等待线程|唤醒所有等待线程|
|**典型场景**|适用于只需要唤醒一个线程的情况|适用于需要唤醒所有线程的情况|

### 示例代码：

#### `pthread_cond_signal` 示例：

```c
#include <pthread.h>
#include <stdio.h>

pthread_cond_t cond = PTHREAD_COND_INITIALIZER;
pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;

void* thread_func(void* arg) {
    pthread_mutex_lock(&mutex);
    printf("Thread %ld waiting on condition...\n", (long)arg);
    pthread_cond_wait(&cond, &mutex);  // 等待条件变量
    printf("Thread %ld resumed\n", (long)arg);
    pthread_mutex_unlock(&mutex);
    return NULL;
}

int main() {
    pthread_t threads[3];
    
    // 创建三个线程
    for (long i = 0; i < 3; i++) {
        pthread_create(&threads[i], NULL, thread_func, (void*)i);
    }

    // 等待一段时间后唤醒一个线程
    pthread_mutex_lock(&mutex);
    printf("Signaling one thread...\n");
    pthread_cond_signal(&cond);  // 唤醒一个线程
    pthread_mutex_unlock(&mutex);

    // 等待线程结束
    for (int i = 0; i < 3; i++) {
        pthread_join(threads[i], NULL);
    }

    return 0;
}
```

#### `pthread_cond_broadcast` 示例：

```c
#include <pthread.h>
#include <stdio.h>

pthread_cond_t cond = PTHREAD_COND_INITIALIZER;
pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;

void* thread_func(void* arg) {
    pthread_mutex_lock(&mutex);
    printf("Thread %ld waiting on condition...\n", (long)arg);
    pthread_cond_wait(&cond, &mutex);  // 等待条件变量
    printf("Thread %ld resumed\n", (long)arg);
    pthread_mutex_unlock(&mutex);
    return NULL;
}

int main() {
    pthread_t threads[3];
    
    // 创建三个线程
    for (long i = 0; i < 3; i++) {
        pthread_create(&threads[i], NULL, thread_func, (void*)i);
    }

    // 等待一段时间后唤醒所有线程
    pthread_mutex_lock(&mutex);
    printf("Broadcasting to all threads...\n");
    pthread_cond_broadcast(&cond);  // 唤醒所有等待线程
    pthread_mutex_unlock(&mutex);

    // 等待线程结束
    for (int i = 0; i < 3; i++) {
        pthread_join(threads[i], NULL);
    }

    return 0;
}
```

### 总结：

- `pthread_cond_signal` 用于唤醒一个等待条件变量的线程。
- `pthread_cond_broadcast` 用于唤醒所有等待条件变量的线程。


