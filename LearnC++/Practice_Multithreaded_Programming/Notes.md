Исключительные теги и заметки
===

## Глава 1

* **Конкурентность** - это переключение задач
* **Пареллелизм** - это использование железа для повышения производительности

## Глава 2

* *std::thread*
```cpp
std::thread t {func};
t.join ();
t.detach ();
t.joinable () == true;
std::thread::hardware_concurrency ();
std::this_thread::get_id ();
std::thread t2;
t2 = std::move (t);
```
* *joinable_thread*

### Глава 3
* *std::mutex mutex;*
* *mutex.lock ();* or *mutex.unlock ()*
* *mutex.try_lock ()*
* *std::lock_guard guard {mutex};*
* *std::lock_guard guard {mutex, std::adopt_lock};*
* *std::scoped_lock guard {mutex};*
* *class threadsafe_stack*
* *std::lock (mutex1, mutex2); std::lock_guard g1 (mutex1); std::lock_guard g2 (mutex2);*
* *std::scoped_lock guard (mutex1, mutex2);*
* *class hierarchical_mutex*
* *std::unique_lock uguard {mutex, std::defer_lock};*
* *uguard.owns_lock (); // Flag*
* Степень детализации блокировок
* *std::call_once*, *std::once_flag*, *std::call_once (init_flag, &X::init_conn, this)*
* *std::shared_mutex*, *std::shared_timed_mutex*
* *std::shared_lock <std::shared_mutex>*
* *std::recursive_mutex*