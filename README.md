# Домашнее задание к занятию «3.2. Coroutines: Scopes, Cancellation, Supervision»

## Вопросы: Cancellation

### Вопрос №1

Отработает ли в этом коде строка `<--`? Поясните, почему да или нет.

```kotlin
fun main() = runBlocking {
    val job = CoroutineScope(EmptyCoroutineContext).launch {
        launch {
            delay(500)
            println("ok") // <--
        }
        launch {
            delay(500)
            println("ok")
        }
    }
    delay(100)
    job.cancelAndJoin()
}
```
#### Ответ
Нет. Потому что корутина отменилась через 100 мс, а в каждой корутине задержка 500 мс, поэтому до вызова указанной строки корутина не доходит, происходит отмена. delay - это suspend функция которая возвращает suspendCancellableCoroutine поэтому на этой функции произошла отмена т.к. отменяя родительский скоуп все дочерние корутины также отменяются благодаря «Структурной конкурентности» STRUCTURED CONCURRENCY с реализованным интерфейсом CancellableContinuation.
***


### Вопрос №2

Отработает ли в этом коде строка `<--`. Поясните, почему да или нет.

```kotlin
fun main() = runBlocking {
    val job = CoroutineScope(EmptyCoroutineContext).launch {
        val child = launch {
            delay(500)
            println("ok") // <--
        }
        launch {
            delay(500)
            println("ok")
        }
        delay(100)
        child.cancel()
    }
    delay(100)
    job.join()
}
```
#### Ответ
Нет. Т.к. корутина child отменилась на 100 мс., а в ней delay 500 мс. И до println очередь не дошла.
***

## Вопросы: Exception Handling

### Вопрос №1

Отработает ли в этом коде строка `<--`. Поясните, почему да или нет.

```kotlin
fun main() {
    with(CoroutineScope(EmptyCoroutineContext)) {
        try {
            launch {
                throw Exception("something bad happened")
            }
        } catch (e: Exception) {
            e.printStackTrace() // <--
        }
    }
    Thread.sleep(1000)
}
```
#### Ответ
Нет, т.к. в launch брошено исключение и оно в нем не поймано, оно не пробрасывается дальше, а вызывает падение корутины, и по структурной конкурентности исключение распространяется вверх по всей иерархии Job объектов и падение и их.
***

### Вопрос №2

Отработает ли в этом коде строка `<--`. Поясните, почему да или нет.

```kotlin
fun main() {
    CoroutineScope(EmptyCoroutineContext).launch {
        try {
            coroutineScope {
                throw Exception("something bad happened")
            }
        } catch (e: Exception) {
            e.printStackTrace() // <--
        }
    }
    Thread.sleep(1000)
}
```
#### Ответ
Да. т.к. scope функция coroutineScope{} пробрасывает исключения от дочерних корутин, вместо распространия по иерархии Job, поэтому try-catch ловит исключение, и приложение не падает.
***

### Вопрос №3

Отработает ли в этом коде строка `<--`. Поясните, почему да или нет.

```kotlin
fun main() {
    CoroutineScope(EmptyCoroutineContext).launch {
        try {
            supervisorScope {
                throw Exception("something bad happened")
            }
        } catch (e: Exception) {
            e.printStackTrace() // <--
        }
    }
    Thread.sleep(1000)
}
```
#### Ответ
Да, т.к. supervisorScope пробрасывает исключения произошедшие в нем или во внутренних корутинах, которые успешно захвачены Try-catch
***

### Вопрос №4

Отработает ли в этом коде строка `<--`. Поясните, почему да или нет.

```kotlin
fun main() {
    CoroutineScope(EmptyCoroutineContext).launch {
        try {
            coroutineScope {
                launch {
                    delay(500)
                    throw Exception("something bad happened") // <--
                }
                launch {
                    throw Exception("something bad happened")
                }
            }
        } catch (e: Exception) {
            e.printStackTrace()
        }
    }
    Thread.sleep(1000)
}
```
#### Ответ
Нет, т.к. в соседней корутине без задержки (delay) произойдет исключение, а scope функция coroutineScope{} пробрасывает исключения от дочерних корутин и в случае если происходит исключение в одной асинхронной задаче все остальные отменяются, текущая корутина отменится на выполнении suspend функции delay(500) в связи с тем что произошло исключение в асинхронной задаче в скоупе coroutineScope.
***

### Вопрос №5

Отработает ли в этом коде строка `<--`. Поясните, почему да или нет.

```kotlin
fun main() {
    CoroutineScope(EmptyCoroutineContext).launch {
        try {
            supervisorScope {
                launch {
                    delay(500)
                    throw Exception("something bad happened") // <--
                }
                launch {
                    throw Exception("something bad happened")
                }
            }
        } catch (e: Exception) {
            e.printStackTrace() // <--
        }
    }
    Thread.sleep(1000)
}
```
#### Ответ
Да, т.к. supervisorScope организовывает Supervison не отменяющей родительской корутины при отмены одной из дочерних, и соответственно асинхронные корутины не отменяются, не распространяет свои исключения "вверх по иерархии" классов Job, но пробрасывает исключения от дочерних корутин.
***

### Вопрос №6

Отработает ли в этом коде строка `<--`. Поясните, почему да или нет.

```kotlin
fun main() {
    CoroutineScope(EmptyCoroutineContext).launch {
        CoroutineScope(EmptyCoroutineContext).launch {
            launch {
                delay(1000)
                println("ok") // <--
            }
            launch {
                delay(500)
                println("ok")
            }
            throw Exception("something bad happened")
        }
    }
    Thread.sleep(1000)
}
```
#### Ответ
Нет, т.к. исключение брошенное в CoroutineScope отменит всю иерархию Job, в т.ч. дочернюю корутину на моменте выполнения suspend функции delay(1000) и prinln() вызван не будет.
***

### Вопрос №7

Отработает ли в этом коде строка `<--`. Поясните, почему да или нет.

```kotlin
fun main() {
    CoroutineScope(EmptyCoroutineContext).launch {
        CoroutineScope(EmptyCoroutineContext + SupervisorJob()).launch {
            launch {
                delay(1000)
                println("ok") // <--
            }
            launch {
                delay(500)
                println("ok")
            }
            throw Exception("something bad happened")
        }
    }
    Thread.sleep(1000)
}
```
#### Ответ
Нет, т.к. в CoroutineScope передан контекст SupervisorJob() с организованным  Supervison, и в нем брошено исключение которое отменяет его и дети этого супервайзера также отменяются в этом случае, в т.ч. корутина на моменте выполнения suspend функции delay(1000) и prinln() вызван не будет.
***