# Report, lesson 26

Name: Danil Sobar Group: PO 25-Z Date: 13.10.2026

## Что работает

Отмечаете то что работает: "почти" не считается

* [x] баг найден и устранен
* [x] TODO 1: текст только из пробелов выдает 422, и текст сохраняется без пробелов
* [x] TODO 2: неизвестный ключ в PATCH выдает 422
* [x] TODO 3: `null` в PATCH выдает 422
* [x] TODO 4 и 5: 404 с `"code": "message_not_found"`
* [x] TODO 6: 409 с `"code": "chat_full"`
* Bruno на перезапущенном сервере: ___ of 13

## 1. Трейсбэк

`DELETE /messages/99` до каких-либо изменений в коде.

```text
  File ".../site-packages/fastapi/routing.py", line ..., in run_endpoint_function
    return await run_in_threadpool(dependant.call, **values)
  File ".../site-packages/starlette/concurrency.py", line ..., in run_in_threadpool
    return await anyio.to_thread.run_sync(func)
  File "main.py", line 43, in delete_message
    store.delete(message_id)                     <- mine
  File "storage.py", line 36, in delete
    self.messages.remove(self.get(message_id))   <- mine
ValueError: list.remove(x): x not in list
```

* Последняя строка значит: `get(99)` не нашел сообщение и вернул `None`, а `list.remove(None)` не нашел `None` в списке и выдал `ValueError`.
* Файл и строку на которую вы посмотрите в первую очередь: `storage.py`, строка 36 (`delete`), затем `get` в строке 20 (`return None`), откуда пришел `None`.
* Статус код который получает клиент: 500 (Internal Server Error).

## 2. Баг

* Какой запрос и о чем он говорит? Запрос `02 Create second message` ожидает `id=2`, а получает `id=1`. Значит счетчик id после первого сообщения снова равен 1.
* Где вы остановили программу (файл и строка)? `storage.py`, строка 24 (`message = Message(id=self.next_id, ...)`), с `breakpoint()` и запуском `fastapi dev main.py --no-reload`.
* Какие `p` комманды вы использовали и что они выдали? Копируйте из терминала.

```text
(Pdb) p self.next_id
1
(Pdb) p len(self.messages)
1
(Pdb) n
(Pdb) n
(Pdb) p self.next_id
1
```

* The wrong line, and your fix:
  * Wrong: `self.next_id = len(self.messages)` (`storage.py`, `add`). Счетчик пересчитывается из длины списка, поэтому после первого сообщения он опять равен 1.
  * Fix: `self.next_id += 1`.