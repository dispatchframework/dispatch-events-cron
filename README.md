# Event Driver: cron
The `cron` event driver fires events on a schedule determined by the cron expression passed to the event driver when created. This allows Dispatch to trigger functions in response to events generated on a scheduled basis.

## Getting Started
This guide assumes that you have a working Dispatch instance and access to the Dispatch CLI configured to work with your Dispatch instance. If you need helping creating a Dispatch instance or configuring your Dispatch CLI see the quickstart guide [quickstart](https://github.com/vmware/dispatch/blob/solo/docs/_front/quickstart.md)

### Create a Function
For this walkthrough we will use the hello.py example available [here](https://github.com/vmware/dispatch/tree/solo/examples/python3). To begin we will seed Dispatch with the images we need for this function. If you followed the instructions in the quickstart guide this step should be done already and should not be repeated.

```bash
$ dispatch create seed-images
```

This will download and create the base images and images for python, java, nodejs, and powershell. Wait for the python image's status to change to READY. You can check this by running the following.
```bash
$ dispatch get images
     NAME    |                         URL                          |    BASEIMAGE    | STATUS |         CREATED DATE
-------------------------------------------------------------------------------------------------------------------------
  java       | dispatch/ddb9cdc6-7727-453a-9d7b-9958ca0102a2:latest | java-base       | READY  | Mon Nov 26 15:38:42 PST 2018
  nodejs     | dispatch/1a3600ba-2aec-4acc-987a-7e4edbf79e3b:latest | nodejs-base     | READY  | Mon Nov 26 15:38:42 PST 2018
  powershell | dispatch/d5f8f5fa-7eca-444f-a9ce-3073495eb2a0:latest | powershell-base | READY  | Mon Nov 26 15:38:42 PST 2018
  python3    | dispatch/e3892573-3105-45b9-b6e1-07be7a607cdd:latest | python3-base    | READY  | Mon Nov 26 15:38:42 PST 2018
  ```

For this example we will use the python hello world example. You can download the python file using curl.
```bash
$ curl https://raw.githubusercontent.com/vmware/dispatch/solo/examples/python3/hello.py > hello.py
```

Now we can create the function. We'll call it hello-py, use the python3 image, the hello.py file we just downloaded, and set the handler to the handle method in hello.py.
```bash
$ dispatch create function hello-py --image python3 --handler hello.handle hello.py
```

Finally, wait for the function status to change to READY. You can use `dispatch exec` to test the function.
```bash
$ dispatch exec hello-py --wait
{
    "blocking": true,
    "executedTime": 1543281864,
    "faasId": "18d394a9-c960-415b-ba91-6cb2c82e5714",
    "finishedTime": 1543281864,
    "functionId": "6a41c89c-0ce8-41ed-8109-229528be8e06",
    "functionName": "hello-py",
    "input": {},
    "logs": {
        "stderr": null,
        "stdout": null
    },
    "name": "69005585-2446-439a-9715-d6ac62a59c22",
    "output": {
        "myField": "Hello, Noone from Nowhere"
    },
    "reason": null,
    "secrets": [],
    "services": null,
    "status": "READY",
    "tags": []
}
```

### Create the Event Driver
Next we need to create the event driver. This requires two steps. First we need to create the event driver type. We can do that with the following command.
```bash
$ dispatch create eventdrivertype cron dispatchframework/dispatch-events-cron:0.0.1
```

Next we can create an instance of the event driver and configure it with our cron spec. Here you can replace the `0/10 * * * *` with any valid cron expression excluding the command to run. The event driver will handle firing an event for you. In this case we are scheduling our function to run at the start of every minute and every ten seconds after that.
```bash
$ dispatch create eventdriver cron --name every-10-seconds --set cron="0/10 * * * *"
```

### Create the Subscription
Now we need to tie the two pieces together. To do that we will create a subscription to trigger a function when events are received from the cron driver.

```bash
$ dispatch create subscription hello-py --name cron-sub --event-type cron.trigger
```

In this example `hello-py` is the name of the function we created previously, and `cron.trigger` is the type of event emitted by the cron driver.

Now we can use `dispatch get runs` to see that our function has been executed.
```bash
$ dispatch get runs
                   ID                  | FUNCTION | STATUS |           STARTED            |           FINISHED
--------------------------------------------------------------------------------------------------------------------
  3e5b6d1c-f48d-4543-b0bd-4a6fe1ae125e | hello-py | READY  | Mon Nov 26 16:35:50 PST 2018 | Mon Nov 26 16:35:51 PST 2018
  c5f1b6f3-7cd8-4cac-a3c0-9b1471e191e5 | hello-py | READY  | Mon Nov 26 16:35:40 PST 2018 | Mon Nov 26 16:35:40 PST 2018
  67298604-6e68-4f8e-a451-39ef78e22a2a | hello-py | READY  | Mon Nov 26 16:35:30 PST 2018 | Mon Nov 26 16:35:30 PST 2018
  04d16d71-ffee-4bd1-8075-15b4462c7f6e | hello-py | READY  | Mon Nov 26 16:35:20 PST 2018 | Mon Nov 26 16:35:20 PST 2018
  2823e9fc-1cf5-4db5-bf47-7ffebd36d840 | hello-py | READY  | Mon Nov 26 16:35:10 PST 2018 | Mon Nov 26 16:35:10 PST 2018
  5ed75128-9dae-44d0-8436-38b3572ed087 | hello-py | READY  | Mon Nov 26 16:35:00 PST 2018 | Mon Nov 26 16:35:00 PST 2018
```

Here you can see the function triggered every 10 seconds. You can now use the cron driver to trigger events on whatever schedule you need.