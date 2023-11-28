---
title: "python-multithreading-multiprocessing"
date: 2023-11-27
---

This post walks through basic code examples for multithreading and multiprocessing in Python. The use case I’ll be discussing is sending and receiving web requests. First, let’s examine a simple program that makes web requests without any sort of multithreading or multiprocessing.


### normal.py
```python
import time
import requests

def make_request(url):
    """Makes a web request and prints the URL and response code."""
    resp = requests.get(url)
    print("Url: {}".format(url))
    print("Response code: {}\n".format(resp.status_code))

if __name__ == '__main__':
    urls = ["https://www.google.com"] * 30
    start = time.time()
    for url in urls:
        make_request(url)
    print("Execution time = {0:.5f}".format(time.time() - start))
```
Here is the output from running the above script.

```
jake@debian-jake:~/$ python3 normal.py
Url: https://www.google.com
Response code: 200

<snip>

Url: https://www.google.com
Response code: 200

Execution time = 8.99106
```
Nine seconds to make 30 web requests doesn’t seem like a long time, but imagine making 30,000 web requests. Introducing multithreading or multiprocessing to this script will be able to greatly increase the requests per second.

## Multithreading

Here is the multithreaded version of the script, which is heavily commented to explain the new constructs that have been introduced. The flow of the program starts at the if __name__ == ‘__main__’ line, so start reading from there if you want to understand the execution flow. After initializing a queue containing URLs that the threads will pull from, the threads are spun up and sent to the manage queue function, which then calls the make_request function.


### multithreaded.py
```python
import threading
import time
from queue import Queue
import requests

def make_request(url):
    """Makes a web request, prints the thread name, URL, and 
    response code.
    """
    resp = requests.get(url)
    with print_lock:
        print("Thread name: {}".format(threading.current_thread().name))
        print("Url: {}".format(url))
        print("Response code: {}\n".format(resp.status_code))

def manage_queue():
    """Manages the url_queue and calls the make request function"""
    while True:

        # Stores the URL and removes it from the queue so no 
        # other threads will use it. 
        current_url = url_queue.get()

        # Calls the make_request function
        make_request(current_url)

        # Tells the queue that the processing on the task is complete.
        url_queue.task_done()

if __name__ == '__main__':

    # Set the number of threads.
    number_of_threads = 5
    
    # Needed to safely print in mult-threaded programs.
    # https://stackoverflow.com/questions/40356200/python-printing-in-multiple-threads
    print_lock = threading.Lock()
    
    # Initializes the queue that all threads will pull from.
    url_queue = Queue()

    # The list of URLs that will go into the queue.
    urls = ["https://www.google.com"] * 30

    # Start the threads.
    for i in range(number_of_threads):

        # Send the threads to the function that manages the queue.
        t = threading.Thread(target=manage_queue)

        # Makes the thread a daemon so it exits when the program finishes.
        t.daemon = True
        t.start()
    
    start = time.time()

    # Puts the URLs in the queue
    for current_url in urls:
        url_queue.put(current_url)

    # Wait until all threads have finished before continuing the program.
    url_queue.join()

    print("Execution time = {0:.5f}".format(time.time() - start))
```
Here is the output from running the above script.

```
jake@debian-jake:~/$ python3 multithreaded.py 
Thread name: Thread-2
Url: https://www.google.com
Response code: 200

Thread name: Thread-1
Url: https://www.google.com
Response code: 200

<snip>

Thread name: Thread-4
Url: https://www.google.com
Response code: 200

Thread name: Thread-3
Url: https://www.google.com
Response code: 200

Execution time = 2.17279
```
So by using 5 threads we reduced the time from 8.99 seconds to 2.17 seconds. While the code is pretty well commented, I’ll go over the need for using a queue.

You can think of a queue as a controlled list. In the first example program we had a list of URLs that were iterated over one at a time and passed to the make_request() function. If this were a human doing work, it would be one human taking a widget off the assembly line and passing it to the widget packaging machine. In the above multithreaded program we use five threads, so it would be like five humans trying to pull the widgets off the assembly line. The five humans would each think they needed to grab the same widget and would do the same work five times instead of splitting the work between five people.

A queue manages which items have been assigned to each thread, and makes sure that each job is only done once. First, the queue must be constructed (url_queue = Queue()). Then, the items must be put into the queue (for current_url in urls:url_queue.put(current_url)). When a thread is ready to do work with an item, they take (remove) an item from a queue (current_url = url_queue.get()). Finally, when the thread has completed its work with the item, they inform the queue the work is complete (url_queue.task_done()).

So in the above example, the actual work takes place in the manage_queue() function. The threads are constructed (for i in range(number_of_threads):), then assigned to run the manage_queue() function (t = threading.Thread(target=manage_queue)), and then started (t.start()). We then make the threads to finish all of the work before continuing the program (url_queue.join()).

Now we take a look at how to implement multiprocessing.

## Multiprocessing

Here is the script with multi-processing implemented.


### multiprocessed.py
```python
from multiprocessing import Pool, cpu_count
import time
import requests

def make_request(url):
    """Makes a web request and prints the URL and response code."""
    resp = requests.get(url)
    print("Url: {}".format(url))
    print("Response code: {}\n".format(resp.status_code))

if __name__ == '__main__':
    urls = ["https://www.google.com"] * 30
    start = time.time()

    # Creates a process pool with n processes, where
    # n is the number returned by cpu_count
    with Pool(cpu_count()) as p:

        # Spawns a process for each URL
        p.map(make_request, urls)
    print("Execution time = {0:.5f}".format(time.time() - start))
```
Here is the output from running the above script.

```
jake@debian-jake:~/$ python3 multiprocessed.py
Url: https://www.google.com
Response code: 200

<snip>

Url: https://www.google.com
Response code: 200

Execution time = 1.64684
```
If this script didn’t work for you, I’ll assume you are running it on Windows. Multiprocessing is implemented slightly different in Windows, specifically regarding the if __name__ == '__main__': part. On Linux, the __name__ of each spawned process will be'__main__'. However, on Windows the __name__ for each spawned process will be'__mp_main__', causing any code in the if block to not execute. To make the code cross compatible, you can move the code outside of that block and it should work fine.

## Multithreading multiple processes

On occasion, you may need to combine multithreading and multiprocessing. Here is a simplified example combining the techiniques.

### multithreaded_multiprocessed.py
```python
from multiprocessing import Pool, cpu_count
import threading
import time
from queue import Queue
import requests

def make_request(url):
    """Makes a web request and prints the URL and response code."""
    resp = requests.get(url)
    print("Url: {}".format(url))
    print("Response code: {}\n".format(resp.status_code))

def manage_queue(url, dir_queue):
    """Manages the dir_queue and calls the make_request function"""
    while True:
        directory = dir_queue.get()
        resource = url.strip('/') + '/' + directory
        make_request(resource)
        dir_queue.task_done()

def do_multithreading(url):
    """Starts the multithreading"""

    # Set the number of threads.
    number_of_threads = 5

    # Initializes the queue.
    dir_queue = Queue()

    # Starts the multithreading
    for i in range(number_of_threads):
        t = threading.Thread(target=manage_queue, args=[url, dir_queue])
        t.daemon = True
        t.start()    

    for directory in words:
        dir_queue.put(directory)
    dir_queue.join()

if __name__ == '__main__':
    urls = ["https://www.google.com", "https://www.bing.com", "https://yahoo.com"]
    words = ["maps", "news", "videos", "foo", "bar",
             "admin", "test", ".htaccess", "search", "blah"]
    start = time.time()

    # Creates a process pool with n processes, where
    # n is the number returned by cpu_count
    with Pool(cpu_count()) as p:

        # Spawns a process for each URL
        p.map(do_multithreading, urls)
    print("Execution time = {0:.5f}".format(time.time() - start))
```
Here is the output:

```
jake@debian-jake:~/$ python3 multithreaded_multiprocessed.py 
Url: https://www.bing.com/foo
Response code: 404

Url: https://www.google.com/foo
Response code: 404

Url: https://yahoo.com/bar
Response code: 404

Url: https://yahoo.com/videos
Response code: 404

<snip>

Url: https://yahoo.com/foo
Response code: 404

Url: https://www.google.com/videos
Response code: 404

Url: https://www.bing.com/bar
Response code: 200

Url: https://yahoo.com/blah
Response code: 404

Url: https://yahoo.com/maps
Response code: 200

Url: https://yahoo.com/search
Response code: 200

Execution time = 2.74469
```
The important thing to note is that all of the specified sites are being force browsed simultaneously with a process devoted to a URL, and each process is using multiple threads to make the web requests with the wordlist.

This combination of multiprocessing and multithreading is incredibly powerful, as it can greatly increase the speed of your programs.
