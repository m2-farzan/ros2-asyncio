# ROS 2 + Asyncio

Geeks. In this post, I want to show you how to use rclpy with asyncio. As you will see, it's easy and will improve your code quality.

## FAQ

- *Why asyncio?*

  Because it's cool.

- *Asyncio is just threads, right?*

  No. While asyncio is often used to achieve concurrency, it doesn't use plain multithreading model.

- *Where to learn about asyncio?*

  I strongly recommend [this video](https://www.youtube.com/watch?v=8aGhZQkoFbQ). It shows stuff in javascript, but the concepts are the same. After learning the fundumentals, you can learn the Python syntax in [this video](https://www.youtube.com/watch?v=F19R_M4Nay4).

- *Wait, but rclpy already has async!*

  While the concept of async is available in rclpy, the implementation is not compatible with the modern async/await semantics.

- *Why did you publish this as a blog post, instead of a library?*

  I'm not a big fan of making libraries for every trivial trick. Let's learn something.

## A trivial async subscriber

Let's start with a simple idea:

```python
import asyncio

import rclpy
from std_msgs.msg import String

rclpy.init()
node = rclpy.create_node('async_subscriber')

async def main():
    print('Node started.')
    async def msg_callback(msg):
        print(f'Message received: {msg}')
    node.create_subscription(String, '/test', msg_callback, 10)
    print('Listening to topic /test...')

async def ros_loop():
    while rclpy.ok():
        rclpy.spin_once(node, timeout_sec=0)
        await asyncio.sleep(1e-4)

if __name__ == "__main__":
    future = asyncio.wait([ros_loop(), main()])
    asyncio.get_event_loop().run_until_complete(future)
```

Note that:

- This code works.

- The `main()` function is so simple and linear. We no longer have to write every program in loops, because now we take care of `spin` in a separate loop.

- The code and callbacks are asyncio-compatible (defined with `async def`), allowing us to use a world of asyncio-compatible libraries.

- The code outside `main()` is boiler plate, and can be copied around with little-to-no modifications.

## A linear subscriber

I don't like callbacks. A linear code is easy to reason about. A code that jumps around is not. Let's see how to avoid callbacks in our main code using asyncio:

```python
import asyncio

import rclpy
from std_msgs.msg import String

rclpy.init()
node = rclpy.create_node('async_subscriber_linear')

async def main():
    print('Node started.')
    async for msg in SubscriptionX(node, String, '/test', 10).messages():
        print(f'Message received: {msg}')

class SubscriptionX:
    message_queue = asyncio.Queue()
    def __init__(self, node, type, topic, qos):
        node.create_subscription(type, topic, self.msg_callback, qos)
    async def msg_callback(self, msg):
        await self.message_queue.put(msg)
    async def messages(self):
        while True:
            yield await self.message_queue.get()

async def ros_loop():
    while rclpy.ok():
        rclpy.spin_once(node, timeout_sec=0)
        await asyncio.sleep(1e-4)

if __name__ == "__main__":
    future = asyncio.wait([ros_loop(), main()])
    asyncio.get_event_loop().run_until_complete(future)
```

Look at that `main()` function! No more callbacks! Of course, we had to write `SubscriptionX` class to handle the callback internally but that's a one-off utility code. Our main code is cleaner now. The `async for` syntax waits for a message to come, and then executes the loop.

Note that I'm just showing that it *can* be done this way, and it *might* be useful in some cases. The question of whether *you* should do it is to be answered by your judgment. Try to imagine scenarios in which this approach would be useful and scenarios in which callbacks would be better.

## A better action client

I personally find rclpy action client API a bit messy. Those get much cleaner using async. In fact, I wrote this post mainly for this section.

Let's define this class:

```python
import asyncio
from rclpy.action import ActionClient

class ActionClientX(ActionClient):
    feedback_queue = asyncio.Queue()

    async def feedback_cb(self, msg):
        await self.feedback_queue.put(msg)

    async def send_goal_async(self, goal_msg):
        goal_future = super().send_goal_async(
            goal_msg,
            feedback_callback=self.feedback_cb
        )
        client_goal_handle = await asyncio.ensure_future(goal_future)
        if not client_goal_handle.accepted:
            raise Exception("Goal rejected.")
        result_future = client_goal_handle.get_result_async()
        while True:
            feedback_future = asyncio.ensure_future(self.feedback_queue.get())
            tasks = [result_future, feedback_future]
            await asyncio.wait(tasks, return_when=asyncio.FIRST_COMPLETED)
            if result_future.done():
                result = result_future.result().result
                yield (None, result)
                break
            else:
                feedback = feedback_future.result().feedback
                yield (feedback, None)
```

Now let's use it in our action client: (For the action server I'm using fibonacci_action_server example from [here](https://docs.ros.org/en/galactic/Tutorials/Actions/Writing-a-Py-Action-Server-Client.html))

```python
import asyncio

import rclpy
from action_tutorials_interfaces.action import Fibonacci

from rclpyx import ActionClientX

rclpy.init()
node = rclpy.create_node('async_action_client')

async def main():
    print('Node started.')
    action_client = ActionClientX(node, Fibonacci, 'fibonacci')
    goal_msg = Fibonacci.Goal()
    goal_msg.order = 10
    async for (feedback, result) in action_client.send_goal_async(goal_msg):
        if feedback:
            print(f'Feedback: {feedback}')
        else:
            print(f'Result: {result}')
    print('Finished.')

async def ros_loop():
    while rclpy.ok():
        rclpy.spin_once(node, timeout_sec=0)
        await asyncio.sleep(1e-4)

if __name__ == "__main__":
    future = asyncio.wait([ros_loop(), main()])
    asyncio.get_event_loop().run_until_complete(future)
```

Look at the `main()` function and compare it to the example in the fibonacci tutorial page.

## Fix exceptions

If you change some line in `main()` into a faulty code, you'll notice that the exception will not be raised when they occur. To fix it, change the last lines of your files to look like this example:

```python
import asyncio

import rclpy

rclpy.init()
node = rclpy.create_node('async_action_client')

async def main():
    print('Node started.')
    print(f'{2/0}')
    print('Finished.')

async def ros_loop():
    while rclpy.ok():
        rclpy.spin_once(node, timeout_sec=0)
        await asyncio.sleep(1e-4)

if __name__ == "__main__":
    future = asyncio.wait(
        [ros_loop(), main()],
        return_when=asyncio.FIRST_EXCEPTION
    )
    done, _pending = asyncio.get_event_loop().run_until_complete(future)
    for task in done:
        task.result()  # raises exceptions if any
```
Now the exception will be raised right away.

## Conclusion

That's it for this post. We've seen how async/await can improve our ROS code. The same approaches can be used for services, lifecycle tools, etc. Async/await is a powerful tool and with a little creativity, you can push what I've shown in this post to the next level. Good news: handling edge cases like exceptions is also easy when using async/await.

In the FAQ section, I argued against making a library for this, probably named rclpy-asyncio. Your view may differ. Maybe you'll build on this and push it to the point that it'd begin to make sense to refactor it and publish the library. If so, it'd be nice to mention this post in the docs.

## Shameless plug

I'm working on a personal project, [Cake Robotics](https://github.com/cakerobotics/crl), which aims to build even more abstraction on top of ROS.

The motivation is that ROS is not technically an OS but it has got the spirit. Now, OS's are good at communicating with hardware, but not so good at being used by programmers. That's why we don't use syscalls in our programming languages.

So the role of Cake Robot Library is to build a layer between ROS and the programmer, so they can write code like the below, and it will Just Work:

```python
import cake

print("Starting the program...")
robot = cake.Robot()

print("Exploring the room...")
robot.navigation.explore(timeout=40)

print("Moving to the starting point...")
robot.navigation.move_to(0, 0)

print("Program finished.")
robot.shutdown()
```

The example above actually works now on Cake! So check out my work, star, contribute, and tell people about it.
