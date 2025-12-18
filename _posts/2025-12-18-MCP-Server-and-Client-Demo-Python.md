FastMCP实现MCP服务端和客户端。<br>
演示服务端返回进度数据（Progress），演示工具执行中途向客户端请求补充信息（Elicitation）。

### 服务端：
```
import asyncio
from fastmcp import FastMCP, Context
from pydantic import BaseModel, Field
class UserInfo(BaseModel):
    name: str = Field(description="any string")
    age: int = Field(description="any integer")
    stream: bool = Field(description="any bool")

mcp = FastMCP(name="mcp-server")
paragraph = """
MCP (Model Context Protocol) is an open-source standard for connecting AI applications to external systems.
"""
@mcp.tool()
async def test_stream(ctx: Context):
    """
    Return character stream of a paragraph
    """
    for chunk in paragraph.split(' '):
        await asyncio.sleep(0.1)
        print(chunk, end="")
        # 返回工具执行过程中的状态，可以实现流式的数据返回
        await ctx.report_progress(1, 2, message=chunk)
    return "DONE"

@mcp.tool()
async def test_elicit(ctx:Context):
    """
    elicitation of mcp
    """
    try:
        # 工具执行过程中，向客户的请求补充信息
        result = await ctx.elicit(
            message="Please provide user information.",
            # 会检查返回的数据，是否符合UserInfo的定义，否则会报错。
            response_type=UserInfo
        )
    except Exception as e:
        print(str(e))
        return str(e)

    print(result)
    # 根据客户端返回时的设置，result.action可能是"accept"/"decline"/"cancel"
    if result.action == "accept":
        print(f"Echo message name = {result.data.name}, age = {result.data.age}, stream = {result.data.stream}")
    elif result.action == "decline":
        return f"Declined!"
    else:  # cancel
        return f"Cancelled!"

    # 客户端elicit返回的数据在result.data中
    # 其数据结构由 response_type 中设置的对象指定
    if result.data.stream:
        await ctx.report_progress(1, 2, message=str(result.data.name))
        await asyncio.sleep(0.2)
        await ctx.report_progress(1, 2, message=str(result.data.age))
        await asyncio.sleep(0.2)
        await ctx.report_progress(1, 2, message=str(result.data.stream))
        await asyncio.sleep(0.2)
    
    print("")
    return f"Echo message name = {result.data.name}, age = {result.data.age}, stream = {result.data.stream}"

def main():
    # make the mcp server run
    mcp.run(
        transport="streamable-http",
        host="0.0.0.0",
        port=9100,
        path="/mcp"
    )
if __name__ == "__main__":
    main()
```

### 客户端：
```
from fastmcp import Client
import asyncio
from fastmcp.client.elicitation import ElicitResult
from mcp.shared.context import RequestContext
from mcp.types import ElicitRequestParams
from pydantic import BaseModel, Field
class UserInfo(BaseModel):
    name: str = Field(description="name of user")
    age: int = Field(description="age of user")
    stream: bool = Field(description="need stream output")

# 服务端请求补充信息时（调用ctx.elicit()），这个函数被调用，
# 在这里可以获得服务端的提示信息（message）
# 要求返回的数据的元数据（response_type）
async def elicitation_handler(  message: str, 
                                response_type: type, 
                                params: ElicitRequestParams, 
                                context: RequestContext):
        print(f"elicitation_handler called:{message}")
        print(response_type.__name__)
        print("")
        print(params)
        print("")
        print(context)
        # You can also require input interactively.
        input_data = UserInfo(name='Henry', age=42, stream=True)
        # Or explicitly return an ElicitResult for more control
        print(f"Return data:{input_data.model_dump()}")
        # 返回的信息封装在ElicitResult对象里
        return ElicitResult(action="accept", content=input_data)

# 服务端返回状态信息时(ctx.report_progress())，这函数被调用,
# progress 表示当前状态在总的状态中的进度
# total 表示总的状态值
# message 是状态信息
async def my_progress_handler(
        progress: float,
        total: float | None,
        message: str | None
    ) -> None:
        print(f"Progress:{progress}, Total:{total}, Message:{message}")

async def main():
    print("--- Creating Client ---")
    # 注册状态回调函数和补充信息回调函数
    client = Client("http://127.0.0.1:9100/mcp",
                    progress_handler=my_progress_handler,
                    elicitation_handler=elicitation_handler)
    try:
        async with client:
            print("--- Client Connected ---")
            print("--- Call Tool test_stream ---")
            result = await client.call_tool("test_stream")
            print("")
            print(result)
            print("--- Call Tool test_elicit ---")
            result = await client.call_tool("test_elicit")
            print("")
            print(result)
    except Exception as e:
        print(f"An error occurred: {e}")
    finally:
        print("--- Client Interaction Finished ---")

if __name__ == "__main__":
    asyncio.run(main())
```
