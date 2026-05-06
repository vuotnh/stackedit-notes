# Một số hàm xử lý cơ bản
1. `asyncio.new_event_loop()`: Tạo và trả về một event loop object.
2. `loop.run_forever()`[](https://docs.python.org/3/library/asyncio-eventloop.html#asyncio.loop.run_forever "Link to this definition"): Run the event loop until  [`stop()`](https://docs.python.org/3/library/asyncio-eventloop.html#asyncio.loop.stop "asyncio.loop.stop")  is called.

	If  [`stop()`](https://docs.python.org/3/library/asyncio-eventloop.html#asyncio.loop.stop "asyncio.loop.stop")  is called before  `run_forever()`  is called, the loop will poll the I/O selector once with a timeout of zero, run all callbacks scheduled in response to I/O events (and those that were already scheduled), and then exit.

	If  [`stop()`](https://docs.python.org/3/library/asyncio-eventloop.html#asyncio.loop.stop "asyncio.loop.stop")  is called while  `run_forever()`  is running, the loop will run the current batch of callbacks and then exit. Note that new callbacks scheduled by callbacks will not run in this case; instead, they will run the next time  `run_forever()`  or  [`run_until_complete()`](https://docs.python.org/3/library/asyncio-eventloop.html#asyncio.loop.run_until_complete "asyncio.loop.run_until_complete")  is called.
3. [`asyncio.run_coroutine_threadsafe(_coro_,  _loop_)`](https://docs.python.org/3/library/asyncio-task.html#asyncio.run_coroutine_threadsafe "Link to this definition"): Submit a coroutine to the given event loop. Thread-safe.

	Return a  [`concurrent.futures.Future`](https://docs.python.org/3/library/concurrent.futures.html#concurrent.futures.Future "concurrent.futures.Future")  to wait for the result from another OS thread.

	This function is meant to be called from a different OS thread than the one where the event loop is running.
4. [`async asyncio.wait_for(aw,  timeout)`](https://docs.python.org/3/library/asyncio-task.html#asyncio.wait_for "Link to this definition"): Wait for the  _aw_  [awaitable](https://docs.python.org/3/library/asyncio-task.html#asyncio-awaitables)  to complete with a timeout.

	If  _aw_  is a coroutine it is automatically scheduled as a Task.

	_timeout_  can either be  `None`  or a float or int number of seconds to wait for. If  _timeout_  is  `None`, block until the future completes.

	If a timeout occurs, it cancels the task and raises  [`TimeoutError`](https://docs.python.org/3/library/exceptions.html#TimeoutError "TimeoutError").

	To avoid the task  [`cancellation`](https://docs.python.org/3/library/asyncio-task.html#asyncio.Task.cancel "asyncio.Task.cancel"), wrap it in  [`shield()`](https://docs.python.org/3/library/asyncio-task.html#asyncio.shield "asyncio.shield").

	The function will wait until the future is actually cancelled, so the total wait time may exceed the  _timeout_. If an exception happens during cancellation, it is propagated.

	If the wait is cancelled, the future  _aw_  is also cancelled.


---
# Mapping code
```python
class MCPToolCallSession(ToolCallSession):
    _ALL_INSTANCES: weakref.WeakSet["MCPToolCallSession"] = weakref.WeakSet()

    def __init__(self, mcp_server: Any, server_variables: dict[str, Any] | None = None) -> None:
        self.__class__._ALL_INSTANCES.add(self)

        self._mcp_server = mcp_server
        self._server_variables = server_variables or {}
        self._queue = asyncio.Queue()
        self._close = False

        self._event_loop = asyncio.new_event_loop()
        self._thread_pool = ThreadPoolExecutor(max_workers=1)
        self._thread_pool.submit(self._event_loop.run_forever)

        asyncio.run_coroutine_threadsafe(self._mcp_server_loop(), self._event_loop)
```
1. Đoạn này khởi tạo một event loop bằng `new_event_loop()`
2. Tạo thread pool và khởi chạy event loop trong sub-thread.
3. Event loop đã chạy nhưng ko có coroutine nào đang chạy.
4. Vì event loop chạy ở thread khác → phải dùng `asyncio.run_coroutine_threadsafe()` để “nhét” coroutine vào loop đó.
### Scheduling from other threads
```python
# Pattern thường dùng để bridge giữa sync world và async world
def in_thread(loop: asyncio.AbstractEventLoop) -> None:
    # Run some blocking IO
    pathlib.Path("example.txt").write_text("hello world", encoding="utf8")

    # Create a coroutine
    coro = asyncio.sleep(1, result=3)

    # Submit the coroutine to a given loop
    future = asyncio.run_coroutine_threadsafe(coro, loop)

    # Wait for the result with an optional timeout argument
    assert future.result(timeout=2) == 3

async def amain() -> None:
    # Get the running loop
    loop = asyncio.get_running_loop()

    # Run something in a thread
    await asyncio.to_thread(in_thread, loop)
```

### Vì sao không `await self._mcp_server_loop()` luôn?
Vì đang ở **thread chính**, còn loop nằm ở **thread khác**

👉 `await` chỉ dùng khi:
-   đang **inside cùng event loop**

Ở đây:

-   Loop A (main thread) ❌
-   Loop B (`self._event_loop` trong thread pool) ✅
👉 Không thể `await` trực tiếp cross-thread

`run_coroutine_threadsafe`: 
-   Nó cho phép **submit coroutine vào loop khác thread**
-   Trả về `Future` thread-safe

```python
async def _mcp_server_loop(self) -> None:
    url = self._mcp_server.url.strip()
    raw_headers: dict[str, str] = self._mcp_server.headers or {}
    headers: dict[str, str] = {}

    for h, v in raw_headers.items():
        nh = Template(h).safe_substitute(self._server_variables)
        nv = Template(v).safe_substitute(self._server_variables)
        if nh.strip() and nv.strip().strip("Bearer"):
            headers[nh] = nv

    if self._mcp_server.server_type == MCPServerType.STREAMABLE_HTTP:
        # Streamable HTTP transport
        try:
            async with streamablehttp_client(url, headers) as (read_stream, write_stream, _):
                async with ClientSession(read_stream, write_stream) as client_session:
                    try:
                        await asyncio.wait_for(client_session.initialize(), timeout=5)
                        logging.info("client_session initialized successfully")
                        await self._process_mcp_tasks(client_session)
                    except asyncio.TimeoutError:
                        msg = f"Timeout initializing client_session for server {self._mcp_server.id}"
                        logging.error(msg)
                        await self._process_mcp_tasks(None, msg)
                    except asyncio.CancelledError:
                        logging.warning(f"STREAMABLE_HTTP MCP session cancelled for server {self._mcp_server.id}")
                        return
        except Exception as e:
            logging.exception(e)
            msg = "Connection failed (possibly due to auth error). Please check authentication settings first"
            await self._process_mcp_tasks(None, msg)

    else:
        await self._process_mcp_tasks(None, f"Unsupported MCP server type: {self._mcp_server.server_type}, id: {self._mcp_server.id}")


async def _process_mcp_tasks(self, client_session: ClientSession | None, error_message: str | None = None) -> None:
    while not self._close:
        try:
            mcp_task, arguments, result_queue = await asyncio.wait_for(self._queue.get(), timeout=1)
        except asyncio.TimeoutError:
            continue
        except asyncio.CancelledError:
            break

        logging.debug(f"Got MCP task {mcp_task} arguments {arguments}")

        r: Any = None

        if not client_session or error_message:
            r = ValueError(error_message)
            try:
                await result_queue.put(r)
            except asyncio.CancelledError:
                break
            continue

        try:
            if mcp_task == "list_tools":
                r = await client_session.list_tools()
            elif mcp_task == "tool_call":
                r = await client_session.call_tool(**arguments)
            else:
                r = ValueError(f"Unknown MCP task {mcp_task}")
        except Exception as e:
            r = e
        except asyncio.CancelledError:
            break

        try:
            await result_queue.put(r)
        except asyncio.CancelledError:
            break
```
Đây là coroutine ***consumer***: coroutine này liên tục lấy task từ trong `self._queue` là task queue để xử lý, sau đó đẩy kết quả vào trong `result_queue`.

```python
async def _call_mcp_server(self, task_type: MCPTaskType, timeout: float | int = 8, **kwargs) -> Any:
    if self._close:
        raise ValueError("Session is closed")

    results = asyncio.Queue()
    await self._queue.put((task_type, kwargs, results))

    try:
        result: CallToolResult | Exception = await asyncio.wait_for(results.get(), timeout=timeout)
        if isinstance(result, Exception):
            raise result
        return result
    except asyncio.TimeoutError:
        raise asyncio.TimeoutError(f"MCP task '{task_type}' timeout after {timeout}s")
    except Exception:
        raise
```
Đây là coroutine ***producer***: coroutine này sẽ đẩy task vào trong `self._queue`, để coroutine worker thực hiện task, sau đó lắng nghe `result_queue` để lấy ra kết quả và trả về cho người dùng.
Các hàm *_call_mcp_tool*, *_get_tools_from_mcp_server* đều gọi tới *_call_mcp_server* để thực thi task (bằng cách truyền vào _task_type_).

```python
def get_tools(self, timeout: float | int = 10) -> list[Tool]:
    if self._close:
        raise ValueError("Session is closed")

    future = asyncio.run_coroutine_threadsafe(self._get_tools_from_mcp_server(timeout=timeout), self._event_loop)
    try:
        return future.result(timeout=timeout)
    except FuturesTimeoutError:
        msg = f"Timeout when fetching tools from MCP server: {self._mcp_server.id} (timeout={timeout})"
        logging.error(msg)
        raise RuntimeError(msg)
    except Exception:
        logging.exception(f"Error fetching tools from MCP server: {self._mcp_server.id}")
        raise
```
Tương tự, đẩy coroutine producer (request) vào event loop, coroutine này sẽ enqueue task vào worker (consumer) đã chạy sẵn.


---
```python
async def close(self) -> None:
    if self._close:
        return

    self._close = True

    while not self._queue.empty():
        try:
            _, _, result_queue = self._queue.get_nowait()
            try:
                await result_queue.put(asyncio.CancelledError("Session is closing"))
            except Exception:
                pass
        except asyncio.QueueEmpty:
            break
        except Exception:
            break

    try:
        self._event_loop.call_soon_threadsafe(self._event_loop.stop)
    except Exception:
        pass

    try:
        self._thread_pool.shutdown(wait=True)
    except Exception:
        pass

    self.__class__._ALL_INSTANCES.discard(self)

def close_sync(self, timeout: float | int = 5) -> None:
    if not self._event_loop.is_running():
        logging.warning(f"Event loop already stopped for {self._mcp_server.id}")
        return

    try:
        future = asyncio.run_coroutine_threadsafe(self.close(), self._event_loop)
        try:
            future.result(timeout=timeout)
        except FuturesTimeoutError:
            logging.error(f"Timeout while closing session for server {self._mcp_server.id} (timeout={timeout})")
        except Exception:
            logging.exception(f"Unexpected error during close_sync for {self._mcp_server.id}")
    except Exception:
        logging.exception(f"Exception while scheduling close for server {self._mcp_server.id}")
```
Mục tiêu của hàm này là đẩy coroutine shutdown vào trong event loop.
Coroutine _close_ sẽ:
1. `while not self._queue.empty()`: lặp qua tất cả các task còn tồn tại trong queue (chưa dc consumer xử lý)
2. `_, _, result_queue = self._queue.get_nowait()`: lấy ngay (ko await) từng task ra khỏi queue, loại bỏ _task_type, _argument_, chỉ giữ lại _result_queue_.
3. `await  result_queue.put(asyncio.CancelledError("Session is closing"))`: Gửi một **exception** vào “hộp thư kết quả” của caller
	*Tại sao cần gửi exception vào trong result queue?*
	- Với mỗi task hoặc  mỗi request ta đang khởi tạo một ***result_queue***.
	- `result: CallToolResult | Exception = await  asyncio.wait_for(results.get(), timeout=timeout)` trong `_call_mcp_server` đang lắng nghe ***result_queue*** đó. Nếu ko đẩy gì vào queue, caller sẽ ***lắng nghe vô hạn***.

4. Sau khi đã emty queue, stop event loop và shutdown thread pool.
---
```python
_ALL_INSTANCES: weakref.WeakSet["MCPToolCallSession"] = weakref.WeakSet()
```
Cho phép quản lý nhiều instance của class `MCPToolCallSession`.

```python
def close_multiple_mcp_toolcall_sessions(sessions: list[MCPToolCallSession]) -> None:
    logging.info(f"Want to clean up {len(sessions)} MCP sessions")

    async def _gather_and_stop() -> None:
        try:
            await asyncio.gather(*[s.close() for s in sessions if s is not None], return_exceptions=True)
        except Exception:
            logging.exception("Exception during MCP session cleanup")
        finally:
            try:
                loop.call_soon_threadsafe(loop.stop)
            except Exception:
                pass

    try:
        loop = asyncio.new_event_loop()
        thread = threading.Thread(target=loop.run_forever, daemon=True)
        thread.start()

        asyncio.run_coroutine_threadsafe(_gather_and_stop(), loop).result()
        thread.join()
    except Exception:
        logging.exception("Exception during MCP session cleanup thread management")

    logging.info(f"{len(sessions)} MCP sessions has been cleaned up. {len(list(MCPToolCallSession._ALL_INSTANCES))} in global context.")
```
-   tạo **event loop tạm thời**
-   chạy song song nhiều `close()` của `MCPToolCallSession`.

👉 loop này chỉ đóng vai trò:

-   orchestrator (điều phối)
-   KHÔNG phải loop chính của session

---
## Trong logic hiện tại vẫn tồn tại Race condition quan trọng
### 💥 Case nguy hiểm:

-   `_process_mcp_tasks` vẫn đang chạy
-   `close()` bắt đầu drain queue

```
consumer đang xử lý → chưa put kết quả vào result_queue, close() đã stop loop
```

💥 → task đang chạy bị “cắt ngang”

### 💥 Cách xử lý:
🔒 **Race được xử lý bằng 3 lớp bảo vệ:**
1.  **Flag `_closing`** → chặn task mới + signal cho worker
2.  **Drain chỉ áp dụng cho task _chưa bị lấy_**
3.  **Cancel + await `_worker_task`** → xử lý triệt để task đang chạy dở
4.  **Chỉ stop loop sau khi worker đã kết thúc**
<!--stackedit_data:
eyJoaXN0b3J5IjpbMzYzNTA5MTY3XX0=
-->