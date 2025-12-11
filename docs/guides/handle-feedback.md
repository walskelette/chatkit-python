# Handle feedback

## Enable feedback actions on the client

Collect thumbs up/down feedback so you can flag broken answers, retrain on good ones, or alert humans. Enable the message actions in the client by setting [`threadItemActions.feedback`](https://openai.github.io/chatkit-js/api/openai/chatkit/type-aliases/threaditemactionsoption/); ChatKit.js renders the controls and sends an `items.feedback` request when a user clicks them.

```tsx
const chatkit = useChatKit({
    // ...
    threadItemActions: {
        feedback: true,
    },
})
```

## Implement `add_feedback` on your server

Override the `add_feedback` method on your server to persist the signal anywhere you like.

```python
from chatkit.server import ChatKitServer
from chatkit.types import FeedbackKind

class MyChatKitServer(ChatKitServer[RequestContext]):
    async def add_feedback(
        self,
        thread_id: str,
        item_ids: list[str],
        feedback: FeedbackKind,
        context: RequestContext,
    ) -> None:
        # Example: write to your analytics/QA store
        await record_feedback(
            thread_id=thread_id,
            item_ids=item_ids,
            sentiment=feedback,
            user_id=context.user_id,
        )
```

`item_ids` can include assistant messages, tool calls, or widgets. If you need to ignore certain items (for example, hidden system prompts), filter them here before recording.

## Send a follow-up message when users leave negative feedback

The client sends an `items.feedback` request when someone clicks the thumbs-down button. The server's `process()` method routes that request to your overridden `add_feedback` method, so you can reply to the user from there without changing `process()` itself.

```python
from datetime import datetime

from chatkit.server import ChatKitServer
from chatkit.types import AssistantMessageContent, AssistantMessageItem, FeedbackKind


class MyChatKitServer(ChatKitServer[RequestContext]):
    async def add_feedback(
        self,
        thread_id: str,
        item_ids: list[str],
        feedback: FeedbackKind,
        context: RequestContext,
    ) -> None:
        if feedback == "negative":
            thread = await self.store.load_thread(thread_id, context=context)
            await self.store.add_thread_item(
                thread_id,
                AssistantMessageItem(
                    id=self.store.generate_item_id("message", thread, context),
                    thread_id=thread_id,
                    created_at=datetime.now(),
                    content=[
                        AssistantMessageContent(
                            text="Sorry about that result. I've flagged it for review."
                        )
                    ],
                ),
                context=context,
            )
```

Clients subscribed to the thread will receive the new assistant message immediately, making it easy to acknowledge the thumbs-down feedback in the chat.
