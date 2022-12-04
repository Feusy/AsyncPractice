# Day 1
# Introduction
* Who is who?
* Who knows what about concurrency?

## Mental model & analogy
Team event with cooking together on an open fire (bogrács)
* resources (data): raw food
* shared memory: pot, kitchen table
* processing (operations): pealing vegetables, slicing onions, dicing potatos, cutting meat, adding to common storage (pot), cooking in the pot, eating :)
* processing units (CPUs): people working on the operations

[The world is full of asynchronous workflow](https://codeopinion.com/the-world-is-full-of-asynchronous-workflow/)

## Basic terms
Which terms come to mind when thinking about our topic?
What's the difference between?
* concurrency -the ability for a (part of a) program to execute (partially) out of order
* asynchronicity/asynchrony - events occur independent of the main program flow
* parallelism - multiple computations happen simultaneously
    * data parallelism
    * task parallelism

* *parallelism*
  * *data parallelism*: performing the same task on the same type of resource by multiple people at the same time
    * e.g. 5 people all pealing onions
  * *task parallelism*: performing a different task on same or different type of resources at the same time
    * e.g. 1 person pealing onions while another person is dicing meat
  * combination is also possible (actually the most typical), e.g. when CPU and GPU cooperate
* *asynchronicity*
  * *synchronous*: a processing unit asks another processing unit to do something and waits (actively) until it finishes to continue with the task
    * e.g. there is one master chef instructing the other people. Gives instruction to slice meat and waits until all meat is finished before being able to tell others to slice onions. (Or slices the meat on his own without involving any other people.)
  * *asynchronous*: a processing unit asks another processing unit to do something and does not wait for it to finish before continuing with its own tasks or asking other processing units to do something else (or the same task on different data)
    * e.g. there is one master chef instructing Person 1 to slice meat and right after that Person 2 to peal onions (without waiting for Person 1 to finish)

## Technical Implementation
Why do you think we need processes?
Why do you think we need threads?

Historically a (Neumann architecture) computer could only execute one series of instructions sequentially. There was no form of concurrency or asynchronicity.
This lead to e.g.
* poor user experience (no listening to music while editing the spreadsheet :)
* poor utilization of resources (CPU is idle waiting for HDD to persist results of calculations)

### Process vs. Thread
* Process model
  * isolation and a form of concurrency (better utilization of hardware resources, user can do more than thing at a time)
  * limitations: no shared memory, IPC is expensive, etc.
  * analogy from Unix world: fork()
* Thread model
  * main difference to processes: shared memory
  * OS and hardware support
  * how does it work?
    * dedicated stack per thread
    * owned by process, but has a "sub-lifecycle" of its owning process
 	* OS schedules each with given priority
* Interesting alternative approach of NodeJS: single-threaded, fully asynchronous

### How to use Threads for Parallelism and asynchronicity

#### Create a Thread for 2 operations running in parallel
Introduce .NET BCL items:
* `Thread` 
* `Thread.Start()`
* `Thread.Join()`

##### Starting Point
```csharp
/// <summary>
/// Console App which reads user input and modifies output according to that user input.
/// Create Thread, Start Thread, Join Thread.
/// Communnicate between threads using shared variables (mention how this differs from IPC when communicating between processes.)
/// </summary>
class Program
{
    /// <summary>
    /// 0. EntryPoint
    /// </summary>
    public static void Main()
    {
        Program p = new();
        p.Run();
    }

    private int _spinnerSteps = 3;
    private bool _exit = false;
    private void Run()
    {
        RunInputPrompt();
        RunSpinner();
    }

    private void RunSpinner()
    {
        int currentStep = 0;
        bool forward = true;

        while (!_exit)
        {
            //Reverse direction if necessary
            if (currentStep <= 0)
                forward = true;
            else if (currentStep >= _spinnerSteps)
                forward = false;

            Console.Clear();
            Console.WriteLine($"Spinner length: {_spinnerSteps}");
            Console.WriteLine(new String('|', currentStep));

            //Advance or step back
            if (forward)
                currentStep++;
            else
                currentStep--;

            //Wait for next iteration
            Thread.Sleep(200);
        }
    }
    private void RunInputPrompt()
    {
        while(!_exit)
        {
            var key = Console.ReadKey(true);

            switch(key.Key)
            {
                case ConsoleKey.Q:
                case ConsoleKey.X:
                case ConsoleKey.Escape:
                    _exit = true;
                    break;
                case ConsoleKey.D1:
                    _spinnerSteps = 1;
                    break;
                case ConsoleKey.D2:
                    _spinnerSteps = 2;
                    break;
                case ConsoleKey.D3:
                    _spinnerSteps = 3;
                    break;
            }
        }
    }
}
```

##### Target
```csharp
/// <summary>
/// Console App which reads user input and modifies output according to that user input.
/// Create Thread, Start Thread, Join Thread.
/// Communnicate between threads using shared variables (mention how this differs from IPC when communicating between processes.)
/// </summary>
class Program
{
    /// <summary>
    /// 0. EntryPoint
    /// </summary>
    public static void Main()
    {
        Program p = new();
        p.Run();
    }

    private int _spinnerSteps = 3;
    private bool _exit = false;
    private void Run()
    {
        Thread spinnerThread = new Thread(RunSpinner);
        Thread inputPromptThread = new Thread(RunInputPrompt);

        inputPromptThread.Start();
        spinnerThread.Start();

        Console.WriteLine("Waiting for threads to finish...");
        inputPromptThread.Join();
        spinnerThread.Join();

        Console.WriteLine("Threads finished");
    }


    private void RunSpinner()
    {
        int currentStep = 0;
        bool forward = true;

        while (!_exit)
        {
            //Reverse direction if necessary
            if (currentStep <= 0)
                forward = true;
            else if (currentStep >= _spinnerSteps)
                forward = false;

            Console.Clear();
            Console.WriteLine($"Spinner length: {_spinnerSteps}");
            Console.WriteLine(new String('|', currentStep));

            //Advance or step back
            if (forward)
                currentStep++;
            else
                currentStep--;

            //Wait for next iteration
            Thread.Sleep(200);
        }
    }
    private void RunInputPrompt()
    {
        while(!_exit)
        {
            var key = Console.ReadKey(true);

            switch(key.Key)
            {
                case ConsoleKey.Q:
                case ConsoleKey.X:
                case ConsoleKey.Escape:
                    _exit = true;
                    break;
                case ConsoleKey.D1:
                    _spinnerSteps = 1;
                    break;
                case ConsoleKey.D2:
                    _spinnerSteps = 2;
                    break;
                case ConsoleKey.D3:
                    _spinnerSteps = 3;
                    break;
            }
        }
    }
}
```



### Drawbacks of Threads
* Consumes resources (OS maintains a list of them, memory for stack, priority queues, checking idleness)
* Takes time to setup (create) and finalize (delete)
* Slows down *overall* system because of scheduling, other administrative OS tasks

Investigate with Process Explorer: memory usage over time, thread and handle count over time

Starting point
```csharp
/// <summary>
/// Console App which shows how much memory is allocated by running a lot of threads
/// </summary>
class Program
{
    /// <summary>
    /// 0. EntryPoint
    /// </summary>
    public static void Main()
    {
        Program p = new();
        p.Run();
    }

    private void Run()
    {
        //TODO: Start 10_000 threads without joining them
    }

    private void CalculatePiAndDumpIt()
    {
        Console.WriteLine(GetPi(50));

        //Keep the thread alive
        Thread.Sleep(TimeSpan.FromDays(1));
    }

    private double GetPi(int digits)
    {
        //Source, goal is simplicity not accuracy http://csharphelper.com/blog/2015/03/in-honor-of-pi-day-3-14-approximate-pi-in-c/

        double pi_over_4 = 0;
        double sign = 1;
        for (int term = 0; term < digits; term++)
        {
            //Console.WriteLine(sign + " / " + (term * 2 + 1) + " = " +
            //    (1.0 / (term * 2 + 1)));
            pi_over_4 += sign / (term * 2 + 1);
            sign *= -1;
        }

        double pi = 4 * pi_over_4;
        return pi;
    }

    double Pi(int precision) => 2 * PiInternal(precision);
    double PiInternal(int precision, int iteration = 1) => iteration >= precision ? 1 : 1 + iteration / (2.0 * iteration + 1) * PiInternal(iteration + 1);
}
```

Goal
```csharp
/// <summary>
/// Console App which shows how much memory is allocated by running a lot of threads
/// </summary>
class Program
{
    /// <summary>
    /// 0. EntryPoint
    /// </summary>
    public static void Main()
    {
        Program p = new();
        p.Run();
    }

    private void Run()
    {
        for(int i = 0; i < 1_000_000_000; i++)
        {
            new Thread(CalculatePiAndDumpIt).Start();
            //Join is missing, but not scope of this excersise
        }
    }

    private void CalculatePiAndDumpIt()
    {
        Console.WriteLine(GetPi(50));

        //Keep the thread alive
        Thread.Sleep(TimeSpan.FromDays(1));
    }

    private double GetPi(int digits)
    {
        //Source, goal is simplicity not accuracy http://csharphelper.com/blog/2015/03/in-honor-of-pi-day-3-14-approximate-pi-in-c/

        double pi_over_4 = 0;
        double sign = 1;
        for (int term = 0; term < digits; term++)
        {
            //Console.WriteLine(sign + " / " + (term * 2 + 1) + " = " +
            //    (1.0 / (term * 2 + 1)));
            pi_over_4 += sign / (term * 2 + 1);
            sign *= -1;
        }

        double pi = 4 * pi_over_4;
        return pi;
    }

    double Pi(int precision) => 2 * PiInternal(precision);
    double PiInternal(int precision, int iteration = 1) => iteration >= precision ? 1 : 1 + iteration / (2.0 * iteration + 1) * PiInternal(iteration + 1);
}
```

#### How the ThreadPool addresses some of these concerns
Goal: demonstrate multi-threading code which never starts or joins any thread manually. Use ThreadPool instead.
Introduce .NET BCL items:
* `ThreadPool`
* `ThreadPool.QueueUserWorkitem()`

Starting point: code above

Goal: replace `new Thread()` with `ThreadPool.QueueUserWorkitem()`
```csharp
/// <summary>
/// Console App which shows how much memory is allocated by running a lot of threads
/// </summary>
class Program
{
    /// <summary>
    /// 0. EntryPoint
    /// </summary>
    public static void Main()
    {
        Program p = new();
        p.Run();
    }

    private void Run()
    {
        for(int i = 0; i < 1000_000_000; i++)
        {
            ThreadPool.QueueUserWorkItem(_ => CalculatePiAndDumpIt(), null);
        }
    }

    private void CalculatePiAndDumpIt()
    {
        Console.WriteLine(GetPi(50));

        //Keep the thread alive
        Thread.Sleep(TimeSpan.FromDays(1));
    }

    private double GetPi(int digits)
    {
        //Source, goal is simplicity not accuracy http://csharphelper.com/blog/2015/03/in-honor-of-pi-day-3-14-approximate-pi-in-c/

        double pi_over_4 = 0;
        double sign = 1;
        for (int term = 0; term < digits; term++)
        {
            //Console.WriteLine(sign + " / " + (term * 2 + 1) + " = " +
            //    (1.0 / (term * 2 + 1)));
            pi_over_4 += sign / (term * 2 + 1);
            sign *= -1;
        }

        double pi = 4 * pi_over_4;
        return pi;
    }

    double Pi(int precision) => 2 * PiInternal(precision);
    double PiInternal(int precision, int iteration = 1) => iteration >= precision ? 1 : 1 + iteration / (2.0 * iteration + 1) * PiInternal(iteration + 1);
}
```

Let's actually join those threads and see the difference!

##### Without Thread Pool ~ 37s
Starting point: code above

Goal: call `Thread.Join()` for all started threads

```csharp
using System.Diagnostics;
/// <summary>
/// Console App which shows how much memory is allocated by running a lot of threads
/// </summary>
class Program
{
    /// <summary>
    /// 0. EntryPoint
    /// </summary>
    public static void Main()
    {
        Program p = new();
        p.Run();
    }

    private void Run()
    {
        List<Thread> allThreads = new();

        Stopwatch sw = Stopwatch.StartNew();

        for (int i = 0; i < 5_000; i++)
        {
            Thread newThread = new(() => CalculatePiAndDumpIt(i));
            allThreads.Add(newThread);

            newThread.Start();
        }

        foreach (var thread in allThreads)
        {
            thread.Join();
        }

        sw.Stop();
        Console.WriteLine($"Total time: {sw.ElapsedMilliseconds:0.000} s");
    }

    private void CalculatePiAndDumpIt(int instance)
    {
        Console.WriteLine($"#{instance:00000000}: {GetPi(50)}");
    }

    private double GetPi(int digits)
    {
        //Source, goal is simplicity not accuracy http://csharphelper.com/blog/2015/03/in-honor-of-pi-day-3-14-approximate-pi-in-c/

        double pi_over_4 = 0;
        double sign = 1;
        for (int term = 0; term < digits; term++)
        {
            //Console.WriteLine(sign + " / " + (term * 2 + 1) + " = " +
            //    (1.0 / (term * 2 + 1)));
            pi_over_4 += sign / (term * 2 + 1);
            sign *= -1;
        }

        double pi = 4 * pi_over_4;
        return pi;
    }

    double Pi(int precision) => 2 * PiInternal(precision);
    double PiInternal(int precision, int iteration = 1) => iteration >= precision ? 1 : 1 + iteration / (2.0 * iteration + 1) * PiInternal(iteration + 1);
}
```

##### With Thread Pool - without synchronization
We don't wait for everything to finish...
We need manual synchronization!

Starting point

```csharp
using System.Diagnostics;
/// <summary>
/// Console App which shows how much memory is allocated by running a lot of threads
/// </summary>
class Program
{
    /// <summary>
    /// 0. EntryPoint
    /// </summary>
    public static void Main()
    {
        Program p = new();
        p.Run();
    }

    private void Run()
    {
        List<Thread> allThreads = new();

        Stopwatch sw = Stopwatch.StartNew();

        for (int i = 0; i < 5_000; i++)
        {
            ThreadPool.QueueUserWorkItem(iteration => CalculatePiAndDumpIt((int)iteration), i);
        }

        sw.Stop();
        Console.WriteLine($"Total time: {sw.ElapsedMilliseconds:0.000} s");
    }

    private void CalculatePiAndDumpIt(int instance)
    {
        Console.WriteLine($"#{instance:00000000}: {GetPi(50)}");
    }

    private double GetPi(int digits)
    {
        //Source, goal is simplicity not accuracy http://csharphelper.com/blog/2015/03/in-honor-of-pi-day-3-14-approximate-pi-in-c/

        double pi_over_4 = 0;
        double sign = 1;
        for (int term = 0; term < digits; term++)
        {
            //Console.WriteLine(sign + " / " + (term * 2 + 1) + " = " +
            //    (1.0 / (term * 2 + 1)));
            pi_over_4 += sign / (term * 2 + 1);
            sign *= -1;
        }

        double pi = 4 * pi_over_4;
        return pi;
    }

    double Pi(int precision) => 2 * PiInternal(precision);
    double PiInternal(int precision, int iteration = 1) => iteration >= precision ? 1 : 1 + iteration / (2.0 * iteration + 1) * PiInternal(iteration + 1);
}
```

But this does not wait for every operation to finish

Goal

Let's fix this and synchronize without `Thread.Join` and use `ManualResetEventSlim˙ instead
```csharp
using System.Diagnostics;
/// <summary>
/// Console App which shows how much memory is allocated by running a lot of threads
/// </summary>
class Program
{
    /// <summary>
    /// 0. EntryPoint
    /// </summary>
    public static void Main()
    {
        Program p = new();
        p.Run();
    }

    private void Run()
    {
        List<ManualResetEventSlim> allResetEvents = new();

        Stopwatch sw = Stopwatch.StartNew();

        for (int i = 0; i < 5_000; i++)
        {
            ManualResetEventSlim resetEvent = new();
            int iteration = i;
            allResetEvents.Add(resetEvent);

            //This is needed to avoid incorrect usage of closure due to async execution(!)
            var inputValues = (iteration, resetEvent);

            ThreadPool.QueueUserWorkItem(inputTuple =>
            {
                var (it, re) = (ValueTuple<int, ManualResetEventSlim>) inputTuple;

                CalculatePiAndDumpIt(it);
                re.Set();
            }, inputValues);
        }

        foreach (var resetEvent in allResetEvents)
        {
            resetEvent.Wait();
            resetEvent.Dispose();
        }

        sw.Stop();
        Console.WriteLine($"Total time: {sw.ElapsedMilliseconds:0.000} s");
    }

    private void CalculatePiAndDumpIt(int instance)
    {
        Console.WriteLine($"#{instance:00000000}: {GetPi(50)}");
    }

    private double GetPi(int digits)
    {
        //Source, goal is simplicity not accuracy http://csharphelper.com/blog/2015/03/in-honor-of-pi-day-3-14-approximate-pi-in-c/

        double pi_over_4 = 0;
        double sign = 1;
        for (int term = 0; term < digits; term++)
        {
            //Console.WriteLine(sign + " / " + (term * 2 + 1) + " = " +
            //    (1.0 / (term * 2 + 1)));
            pi_over_4 += sign / (term * 2 + 1);
            sign *= -1;
        }

        double pi = 4 * pi_over_4;
        return pi;
    }

    double Pi(int precision) => 2 * PiInternal(precision);
    double PiInternal(int precision, int iteration = 1) => iteration >= precision ? 1 : 1 + iteration / (2.0 * iteration + 1) * PiInternal(iteration + 1);
}
```

#### Remaining problems
Logically Async code (e.g. IO) still consumes unnecessary threads!

##### 1st try
Fully synchronous, 1 thread, slow
--> Draw the series of operations on a whiteboard (1 main thread executing all along)

Starting point
```csharp
using System.Diagnostics;
/// <summary>
/// Console App which simulates (!) long-running synchronous IO operations
/// </summary>
class Program
{
    /// <summary>
    /// 0. EntryPoint
    /// </summary>
    public static void Main()
    {
        Program p = new();
        p.Run();
    }

    const int COST_OF_IO_IN_MS = 500;
    const int COST_OF_CALC_IN_MS = 500;

    const int NUMBER_OF_FILES = 10;

    private void Run()
    {
        Stopwatch sw = Stopwatch.StartNew();

        //Load 1000 files from simulated file system
        //TODO
        
        sw.Stop();
        Console.WriteLine($"Total time for {NUMBER_OF_FILES} files: {sw.ElapsedMilliseconds:0.000} s");
    }

    private void LoadAndParseFileContents(string filePath)
    {
        Console.WriteLine($"Loading file {filePath}...");
        Byte[] fileContents = LoadFileContents(filePath);

        Console.WriteLine($"Parsing file {filePath}...");
        string result = Parse(fileContents);

        Console.WriteLine($"Parsed contents of file {filePath}: {result}");
    }

    #region external code simulated
    private Byte[] LoadFileContents(string filePath)
    {
        //Very expensive IO-bound operation to parse
        Thread.Sleep(COST_OF_IO_IN_MS);
        return new byte[1024];
    }

    private Random _random = new();
    private string Parse(Byte[] contents)
    {
        //Very expensive, CPU-bound operation to parse
        Thread.Sleep(COST_OF_CALC_IN_MS);
        return $"Summary of contents: {(_random.NextSingle() * 10):0}";
    }
    #endregion
}
```

Goal
```csharp
using System.Diagnostics;
/// <summary>
/// Console App which simulates (!) long-running synchronous IO operations
/// </summary>
class Program
{
    /// <summary>
    /// 0. EntryPoint
    /// </summary>
    public static void Main()
    {
        Program p = new();
        p.Run();
    }

    const int COST_OF_IO_IN_MS = 500;
    const int COST_OF_CALC_IN_MS = 500;

    const int NUMBER_OF_FILES = 10;

    private void Run()
    {
        Stopwatch sw = Stopwatch.StartNew();

        //Load 1000 files from simulated file system
        for(int i = 0; i < NUMBER_OF_FILES; i++)
        {
            string filePath = Path.Combine(Path.GetTempPath(), i.ToString("0000") + ".xml");
            LoadAndParseFileContents(filePath);
        }
        

        sw.Stop();
        Console.WriteLine($"Total time for {NUMBER_OF_FILES} files: {sw.ElapsedMilliseconds:0.000} s");
    }

    private void LoadAndParseFileContents(string filePath)
    {
        Console.WriteLine($"Loading file {filePath}...");
        Byte[] fileContents = LoadFileContents(filePath);

        Console.WriteLine($"Parsing file {filePath}...");
        string result = Parse(fileContents);

        Console.WriteLine($"Parsed contents of file {filePath}: {result}");
    }

    #region external code simulated
    private Byte[] LoadFileContents(string filePath)
    {
        //Very expensive IO-bound operation to parse
        Thread.Sleep(COST_OF_IO_IN_MS);
        return new byte[1024];
    }

    private Random _random = new();
    private string Parse(Byte[] contents)
    {
        //Very expensive, CPU-bound operation to parse
        Thread.Sleep(COST_OF_CALC_IN_MS);
        return $"Summary of contents: {(_random.NextSingle() * 10):0}";
    }
    #endregion
}
```

##### 2nd try
Each operation on a threadpool thread, faster, multiple threads blocked for IO
--> Draw the series of operations on a whiteboard (1 main thread + as many threads as many files / thread pool allows)

Introduce BCL type `CountDownEvent` for simpler synchronization

Starting point: previous code

Goal
```csharp
using System.Diagnostics;
/// <summary>
/// Console App which simulates (!) long-running synchronous IO operations
/// </summary>
class Program
{
    /// <summary>
    /// 0. EntryPoint
    /// </summary>
    public static void Main()
    {
        Program p = new();
        p.Run();
    }

    const int COST_OF_IO_IN_MS = 500;
    const int COST_OF_CALC_IN_MS = 500;

    const int NUMBER_OF_FILES = 10;

    private void Run()
    {
        Stopwatch sw = Stopwatch.StartNew();

        //Introduce a simpler sync mechanism when possible
        //Be aware of IDisposable and don't forget to "using" it :)
        using CountdownEvent countDownEvent = new(NUMBER_OF_FILES);

        //Load NUMBER_OF_FILES files from simulated file system
        for (int i = 0; i < NUMBER_OF_FILES; i++)
        {
            string filePath = Path.Combine(Path.GetTempPath(), i.ToString("0000") + ".xml");
            ThreadPool.QueueUserWorkItem(fileName =>
            {
                LoadAndParseFileContents(filePath);
                countDownEvent.Signal();
            }, filePath);
        }

        Console.WriteLine("Waiting for everyone to finish...");
        countDownEvent.Wait();

        sw.Stop();
        Console.WriteLine($"Total time for {NUMBER_OF_FILES} files: {sw.ElapsedMilliseconds:0.000} s");
    }

    private void LoadAndParseFileContents(string filePath)
    {
        Console.WriteLine($"Loading file {filePath}...");
        Byte[] fileContents = LoadFileContents(filePath);

        Console.WriteLine($"Parsing file {filePath}...");
        string result = Parse(fileContents);

        Console.WriteLine($"Parsed contents of file {filePath}: {result}");
    }

    #region external code simulated
    private Byte[] LoadFileContents(string filePath)
    {
        //Very expensive IO-bound operation to parse
        Thread.Sleep(COST_OF_IO_IN_MS);
        return new byte[1024];
    }

    private Random _random = new();
    private string Parse(Byte[] contents)
    {
        //Very expensive, CPU-bound operation to parse
        Thread.Sleep(COST_OF_CALC_IN_MS);
        return $"Summary of contents: {(_random.NextSingle() * 10):0}";
    }
    #endregion
}
```

##### 3rd try
Real asynchronous operations, no threads with blocking IO
--> Draw the series of operations on a whiteboard (1 main thread + no thread per IO operation, discuss "there is no thread" concept)

Starting point: previous code

Goal:

```csharp
using System.Diagnostics;
/// <summary>
/// Console App which simulates (!) long-running synchronous IO operations
/// </summary>
class Program
{
    /// <summary>
    /// 0. EntryPoint
    /// </summary>
    public static void Main()
    {
        Program p = new();
        p.Run();
    }

    const int COST_OF_IO_IN_MS = 500;
    const int COST_OF_CALC_IN_MS = 500;

    const int NUMBER_OF_FILES = 1000;

    private void Run()
    {
        Stopwatch sw = Stopwatch.StartNew();

        //Back to ManualResetEventSlim for simplicity
        List<ManualResetEventSlim> allResetEvents = new();

        //Load NUMBER_OF_FILES files from simulated file system
        for (int i = 0; i < NUMBER_OF_FILES; i++)
        {
            ManualResetEventSlim resetEventForCompletionOfTask = new();

            string filePath = Path.Combine(Path.GetTempPath(), i.ToString("0000") + ".xml");
            LoadFileContents(filePath, resetEventForCompletionOfTask, OutputFileContents);

            allResetEvents.Add(resetEventForCompletionOfTask);
        }

        Console.WriteLine("Waiting for everyone to finish...");
        foreach(var resetEvent in allResetEvents)
        {
            resetEvent.Wait();
        }

        sw.Stop();
        Console.WriteLine($"Total time for {NUMBER_OF_FILES} files: {sw.Elapsed.TotalSeconds:0.000} s");
    }

    private void OutputFileContents(Byte[] fileContents)
    {
        Console.WriteLine($"Summary of file contents: {(_random.NextDouble() * 10):0}");
    }

    #region external code simulated
    private void LoadFileContents(string filePath, ManualResetEventSlim signalWhenDone, Action<Byte[]> continuation)
    {
        //Very expensive IO-bound operation to parse - but not using a thread
        //not the point of the excersise to understand this, used only for simulation purposes
        Task.Delay(COST_OF_IO_IN_MS)
            .ContinueWith(_ => continuation(new byte[1024]))
            .ContinueWith(_ => signalWhenDone.Set());
    }

    private Random _random = new();
    private string Parse(Byte[] contents)
    {
        //Very expensive, CPU-bound operation to parse
        Thread.Sleep(COST_OF_CALC_IN_MS);
        return $"Summary of contents: {(_random.NextSingle() * 10):0}";
    }
    #endregion
}
```

Show how this reduces number of threads and why.

Some theory: IO completion ports and Kernel-level abstraction of IO completion

#### The first solution to this problem: Event-Based Async Pattern
Just show some code how freakin' ugly this used to be before Promises (Tasks).
Maybe debug through together?

Starting point (no coding together, just debugging):
```csharp
using System.Diagnostics;
/// <summary>
/// Console App which simulates (!) long-running synchronous IO operations
/// </summary>
class Program
{
    /// <summary>
    /// 0. EntryPoint
    /// </summary>
    public static void Main()
    {
        Program p = new();
        p.Run();
    }

    const int NUMBER_OF_FILES = 1000;

    private void Run()
    {
        Stopwatch sw = Stopwatch.StartNew();

        //Back to ManualResetEventSlim for simplicity
        List<ManualResetEventSlim> allResetEvents = new();

        //Load NUMBER_OF_FILES files from simulated file system
        for (int i = 0; i < NUMBER_OF_FILES; i++)
        {
            ManualResetEventSlim resetEventForCompletionOfTask = new();
            allResetEvents.Add(resetEventForCompletionOfTask);

            string filePath = Path.Combine(Path.GetTempPath(), i.ToString("0000") + ".xml");

            //Load File contents async
            using FileStream fs = new FileStream(filePath, FileMode.OpenOrCreate);
            byte[] buffer = new byte[0];

            //Ugly APM/EAP model (note: example is not fully correct - buffer is part of captured closure)
            var _ = fs.BeginRead(buffer, 0, 0,
                stateObject =>
                {
                    int numberOfBytesSuccessfullyRead = fs.EndRead(stateObject);
                    Console.WriteLine($"Bytes read from file: {numberOfBytesSuccessfullyRead:0}");
                    ((ManualResetEventSlim)stateObject.AsyncState).Set(); //marks overall completion
                },
                resetEventForCompletionOfTask);
        }

        Console.WriteLine("Waiting for everyone to finish...");
        foreach (var resetEvent in allResetEvents)
        {
            resetEvent.Wait();
        }

        sw.Stop();
        Console.WriteLine($"Total time for {NUMBER_OF_FILES} files: {sw.Elapsed.TotalSeconds:0.000} s");
    }
}
```

#### The second solution to this problem - Promise(s) / .NET Tasks
This is a significant mental improvement and understandability over continuations as it carries together:
* state of operation (not started, running, finished, faulted)
* result (return value, if available)
* error information (exceptions, etc)
* possibility to attach further continuations

Starting point
```csharp
using System.Diagnostics;
/// <summary>
/// Console App which simulates (!) long-running synchronous IO operations
/// </summary>
class Program
{
    /// <summary>
    /// 0. EntryPoint
    /// </summary>
    public static void Main()
    {
        Program p = new();
        p.Run();
    }

    const int COST_OF_IO_IN_MS = 500;
    const int COST_OF_CALC_IN_MS = 500;

    const int NUMBER_OF_FILES = 1000;

    private void Run()
    {
        Stopwatch sw = Stopwatch.StartNew();

        //Load NUMBER_OF_FILES files from simulated file system
        for (int i = 0; i < NUMBER_OF_FILES; i++)
        {
            //TODO: call Async Method
        }

        Console.WriteLine("Waiting for everyone to finish...");

        //TODO: wait for all async operations to finish
        //TODO: output contents

        sw.Stop();
        Console.WriteLine($"Total time for {NUMBER_OF_FILES} files: {sw.Elapsed.TotalSeconds:0.000} s");
    }

    private void OutputFileContents(Byte[] fileContents)
    {
        Console.WriteLine($"Summary of file contents: {(_random.NextDouble() * 10):0}");
    }

    #region external code simulated
    private Task<byte[]> LoadFileContentsAsync(string filePath)
    {
        //Very expensive IO-bound operation to parse - but not using a thread
        //not the point of the excersise to understand this, used only for simulation purposes
        return Task.Delay(COST_OF_IO_IN_MS)
                   .ContinueWith(_ => new byte[0]);
    }

    private Random _random = new();
    private string Parse(Byte[] contents)
    {
        //Very expensive, CPU-bound operation to parse
        Thread.Sleep(COST_OF_CALC_IN_MS);
        return $"Summary of contents: {(_random.NextSingle() * 10):0}";
    }
    #endregion
}
```


```csharp
using System.Diagnostics;
/// <summary>
/// Console App which simulates (!) long-running synchronous IO operations
/// </summary>
class Program
{
    /// <summary>
    /// 0. EntryPoint
    /// </summary>
    public static void Main()
    {
        Program p = new();
        p.Run();
    }

    const int COST_OF_IO_IN_MS = 500;
    const int COST_OF_CALC_IN_MS = 500;

    const int NUMBER_OF_FILES = 1000;

    private void Run()
    {
        Stopwatch sw = Stopwatch.StartNew();
        List<Task<byte[]>> allTasks = new();

        //Load NUMBER_OF_FILES files from simulated file system
        for (int i = 0; i < NUMBER_OF_FILES; i++)
        {
            string filePath = Path.Combine(Path.GetTempPath(), i.ToString("0000") + ".xml");
            var fileLoadPromise = LoadFileContentsAsync(filePath);
            allTasks.Add(fileLoadPromise);
        }

        Console.WriteLine("Waiting for everyone to finish...");

        foreach (var task in allTasks)
        {
            task.Wait();
            var result = task.Result;
            OutputFileContents(result);
        }

        sw.Stop();
        Console.WriteLine($"Total time for {NUMBER_OF_FILES} files: {sw.Elapsed.TotalSeconds:0.000} s");
    }

    private void OutputFileContents(Byte[] fileContents)
    {
        Console.WriteLine($"Summary of file contents: {(_random.NextDouble() * 10):0}");
    }

    #region external code simulated
    private Task<byte[]> LoadFileContentsAsync(string filePath)
    {
        //Very expensive IO-bound operation to parse - but not using a thread
        //not the point of the excersise to understand this, used only for simulation purposes
        return Task.Delay(COST_OF_IO_IN_MS)
                   .ContinueWith(_ => new byte[0]);
    }

    private Random _random = new();
    private string Parse(Byte[] contents)
    {
        //Very expensive, CPU-bound operation to parse
        Thread.Sleep(COST_OF_CALC_IN_MS);
        return $"Summary of contents: {(_random.NextSingle() * 10):0}";
    }
    #endregion
}
```

#### Summary: Thread vs. Promise (Task)
* Thread: always consumes dedicated resources (OS object/handle, ~min. 1 MB of memory for stack, which needs clean-up, etc.)
* Promise: just a "pointer" to the finishing of an operation, doesn't consume as much resources as a full-blown thread

#### Some further toying around with Tasks
##### Attaching continuations & error handling

[Source at MSDN](https://docs.microsoft.com/en-us/dotnet/standard/parallel-programming/chaining-tasks-by-using-continuation-tasks)

Starting point
```csharp
using System.Diagnostics;
/// <summary>
/// Console App which toys around with Tasks in general
/// </summary>
class Program
{
    /// <summary>
    /// 0. EntryPoint
    /// </summary>
    public static void Main()
    {
        Program p = new();
        p.Run();
    }

    private void Run()
    {
        //Simple continuation
        //TODO

        //Multiple continuations
        //TODO

        //Continuations depending on success
        //TODO

        //Different continuation strategy
        //TODO
    }

    #region external async methods
    Task DoOperation1Async() => Task.Delay(500);

    Task DoOperation2Async() => Task.Delay(300);

    Task DoOperation3Async(bool throwAnError = false) => throwAnError ? Task.FromException(new Exception("Just a sample exception :)")) : Task.FromResult<object>(null);
    #endregion
}
```

Goal
```csharp
using System.Diagnostics;
/// <summary>
/// Console App which toys around with Tasks in general
/// </summary>
class Program
{
    /// <summary>
    /// 0. EntryPoint
    /// </summary>
    public static void Main()
    {
        Program p = new();
        p.Run();
    }

    private void Run()
    {
        //Simple continuation
        Console.WriteLine("Starting Operation 1...");
        DoOperation1Async()
            .ContinueWith(t =>
            {
                Console.WriteLine($"  Finished Operation 1 {t.Id}");
            }).Wait();

        //Multiple continuations
        Console.WriteLine("Starting Operation 2...");
        DoOperation2Async()
            .ContinueWith(t => Console.WriteLine($"  Finished Operation 2 {t.Id}"))
            .ContinueWith(t => DoOperation3Async(false)
                .ContinueWith(t => Console.WriteLine($"    Finished Operation 3 {t.Id}")))
            .Wait();

        //Continuations depending on success
        Console.WriteLine("Starting Operation 3...");
        DoOperation3Async(true)
            .ContinueWith(t => Console.WriteLine($"  Operation 3 {t.Id} faulted with error: {t.Exception}"), TaskContinuationOptions.OnlyOnFaulted)
            .ContinueWith(t => Console.WriteLine($"  Operation 3 {t.Id} finished successfully."), TaskContinuationOptions.NotOnFaulted)
            .Wait();

        //Different continuation strategy
        Console.WriteLine("Starting Operation 3 with different continuation strategy...");
        var result = DoOperation3Async(true)
            .ContinueWith(t =>
            {
                if (t.IsFaulted)
                {
                    //Console.WriteLine($"  Operation 3 {t.Id} is Faulted with error: {t.Exception}");
                    return false;
                }
                else
                {
                    //Console.WriteLine($"  Operation 3 {t.Id} succeeded");
                    return true;
                }
            })
            .Result;

        if (result == false)
        {
            Console.WriteLine("Operation 3 faulted");
        }
        else
        {
            Console.WriteLine("Operation 3 succeeded");
        }


    }

    #region external async methods
    Task DoOperation1Async() => Task.Delay(500);

    Task DoOperation2Async() => Task.Delay(300);

    Task DoOperation3Async(bool throwAnError = false) => throwAnError ? Task.FromException(new Exception("Just a sample exception :)")) : Task.FromResult<object>(null);
    #endregion

}
```

Re-write with `async` / `await` to make it more readable

```csharp
using System.Diagnostics;
/// <summary>
/// Console App which toys around with Tasks in general
/// </summary>
class Program
{
    /// <summary>
    /// 0. EntryPoint
    /// </summary>
    public static /*async Task*/ void Main()
    {
        Program p = new();
        p.RunAsync().Wait();
    }

    private async Task RunAsync()
    {
        //Simple continuation
        Console.WriteLine("Starting Operation 1...");
        await DoOperation1Async();
        Console.WriteLine($"  Finished Operation 1 ");

        //Multiple continuations
        Console.WriteLine("Starting Operation 2...");
        await DoOperation2Async();
        Console.WriteLine($"  Finished Operation 2 ");
        await DoOperation3Async(false);
        Console.WriteLine($"    Finished Operation 3 ");

        //Continuations depending on success
        Console.WriteLine("Starting Operation 3...");
        try
        {
            await Task.WhenAll(DoOperation3Async(true), DoOperation1Async());
            return;
        }
        catch (Exception ex)
        {
            Console.WriteLine($"  Operation 3 faulted with error: {ex}");
            return;
        }
    }

    #region external async methods
    Task DoOperation1Async() => Task.Delay(500);

    Task DoOperation2Async() => Task.Delay(300);

    Task DoOperation3Async(bool throwAnError = false) => throwAnError ? Task.FromException(new Exception("Just a sample exception :)")) : Task.FromResult<object>(null);
    #endregion

}
```

Explan what async/await really does
* (capture SyncContext - if exists)
* return execution to caller method immediately
* schedule rest of method as a continuation (incl. variables, etc.)
* (schedule continuation on captured SyncContext)
* if async task ran to Exception, Unwrap it and Throw as if it were thrown at the `await` point
* [https://learn.microsoft.com/en-us/dotnet/csharp/programming-guide/concepts/async/task-asynchronous-programming-model](there is no threading involved at all (!))


Why doesn't this work?
(Explain `Task.Factory.StartNew()` )

```csharp
using System.Diagnostics;
/// <summary>
/// Console App which demonstrated TPL error handling
/// </summary>
class Program
{
    /// <summary>
    /// 0. EntryPoint
    /// </summary>
    public static void Main()
    {
        Program p = new();
        p.Run();
    }

    private void Run()
    {
        //The not-working idea
        try
        {
            Task.Factory.StartNew(() => throw new Exception("Just a sample exception :)"));
        }
        catch (Exception ex)
        {
            Console.WriteLine(ex.ToString());
        }
    }
}
```

If there is no continuation attached, result retrieved or Wait()-ed TPL unhandled exception is thrown (!)

Starting point: previous code

Goal:
```csharp
using System.Diagnostics;
/// <summary>
/// Console App which demonstrated TPL error handling
/// </summary>
class Program
{
    /// <summary>
    /// 0. EntryPoint
    /// </summary>
    public static void Main()
    {
#if DEBUG
        if (Debugger.IsAttached)
        {
            for (int i = 0; i < 10; i++)
                Console.WriteLine("**** **** **** Don't forget to run me in *Release* mode without debugging **** **** ****");
        }
#endif

        Program p = new();
        p.Run();
    }

    private void Run()
    {
        TaskScheduler.UnobservedTaskException += TaskScheduler_UnobservedTaskException;

        RunTask();

        Thread.Sleep(1000);

        //This is needed to ensure that Finalizer is called
        GC.Collect();
        GC.WaitForPendingFinalizers();
        GC.Collect();

        Console.WriteLine("Waiting for unhandled Exception reporter");
        Console.ReadKey();
    }

    private void RunTask()
    {
        Task.Run(() => throw new Exception("Just a sample exception :)"));
        //do nothing with *task*
    }

    private void TaskScheduler_UnobservedTaskException(object? sender, UnobservedTaskExceptionEventArgs e)
    {
        Console.WriteLine($"Unobserved exception in TPL: {e.Exception}");
    }
}
```

[Some further details](https://tpodolak.com/blog/2015/08/10/tpl-exception-handling-and-unobservedtaskexception-issue/)

##### Cancellation & Progress Reporting
Show how to properly cancel a potentially long-running operation

`IProgress` interface
* when exposing a method on an interface
* implementing `IProgress` as a consumer of such an interface

The recomenndation for an Async API
Putting it all together:
* `async Task<Result> DoLongRunningOperationAsync(var inputParameters, CancellationToken ct, IProgress<ProgressArgs> iprogress)`

Starting point
```csharp
using System.Diagnostics;
/// <summary>
/// Console App which demonstrates a library that uses all TPL recommendations
/// Async Task, return value, IProgress and CancellationToken (s)
/// </summary>
class Program
{
    /// <summary>
    /// 0. EntryPoint
    /// </summary>
    public static async Task Main()
    {
        Program p = new();
        await p.RunAsync();
    }

    private async Task RunAsync()
    {
        //Get All files to Parse
        string[] allFilesToParse = Directory.GetFiles(Path.GetTempPath())[0..19]; //take only the first 20 items

        try
        {
            //TODO call ParseAllFilesAsync API
            Console.WriteLine("Finished. Good bye");

        }
        catch (OperationCanceledException e)
        {
            Console.WriteLine("Operation was cancelled, results are probably incomplete. Press any key to exit now");
            Console.ReadKey(true);
        }
    }

    private async Task<string> ParseAllFilesAsync(string[] allFilesToParse, IProgress<string>? progressReporter = default, CancellationToken? token = default)
    {
        //TODO: Fill implementation
    }

    private void TaskScheduler_UnobservedTaskException(object? sender, UnobservedTaskExceptionEventArgs e)
    {
        Console.WriteLine($"Unobserved exception in TPL: {e.Exception}");
    }
}
```

Goal
```csharp
using System.Diagnostics;
/// <summary>
/// Console App which demonstrates a library that uses all TPL recommendations
/// Async Task, return value, IProgress and CancellationToken (s)
/// </summary>
class Program
{
    /// <summary>
    /// 0. EntryPoint
    /// </summary>
    public static async Task Main()
    {
        Program p = new();
        await p.RunAsync();
    }

    private async Task RunAsync()
    {
        //Get All files to Parse
        string[] allFilesToParse = Directory.GetFiles(Path.GetTempPath())[0..19]; //take only the first 20 items

        //Subscribe to CTRL+C
        CancellationTokenSource cts = new();
        Console.CancelKeyPress += (_, args) =>
        {
            if (!cts.IsCancellationRequested)
            {
                //this is the 1st time user pressed CTRL+C so we shouldn't yet exit
                args.Cancel = true;

                cts.Cancel();
            }
            else
            {
                //this is the 2nd time user pressed CTRL+C so we can now exit
                args.Cancel = false;
            }
        };

        //Do progress reporting
        IProgress<string> progressReporter = new ConsoleProgressReporter();

        try
        {
            string result = await ParseAllFilesAsync(allFilesToParse, progressReporter, cts.Token);
            Console.WriteLine("Finished. Good bye");

        }
        catch(OperationCanceledException e)
        {
            Console.WriteLine("Operation was cancelled, results are probably incomplete. Press any key to exit now");
            Console.ReadKey(true);
        }
    }

    private async Task<string> ParseAllFilesAsync(string[] allFilesToParse, IProgress<string>? progressReporter = default, CancellationToken? token = default)
    {
        token?.ThrowIfCancellationRequested();

        int count = 0;

        foreach(var file in allFilesToParse)
        {
            token?.ThrowIfCancellationRequested();

            progressReporter?.Report($"Analyzing file #{count} {file}...");
            await Task.Delay(500);
            progressReporter?.Report($"Analyzed  file #{count} {file}");

            count++;
        }

        progressReporter?.Report($"Done");

        return $"Analyzed {count + 1} files";
    }

    private void TaskScheduler_UnobservedTaskException(object? sender, UnobservedTaskExceptionEventArgs e)
    {
        Console.WriteLine($"Unobserved exception in TPL: {e.Exception}");
    }
}

internal class ConsoleProgressReporter : IProgress<string>
{
    public void Report(string value)
    {
        Console.WriteLine($"Progress at {DateTime.Now}: {value}");
    }
}
```

# Day 2
## When to use Parallelism and when to use Asynchronitiy?
* Reasons for parallelism
    * utilize more (logical) cores to get (compute-intensive and *not* IO) work done faster
    * usually combined with some form of asynchronicity ("main" thread requests work from "slave" threads and synchronizes them at the end)
* Reasons for asynchronicity
    1. Frontend: free *the* UI context (thread) from long-running operations (IO or CPU-intensive work)
    2. Backend: do not waste threads waiting for IO to complete
        * mention again NodeJS event-driven, asynchronous, single-threaded (!) approach
        * there is usually no other reason to use asynchronicity on the backend!
        (exceptions might be custom single-threaded contexts such as e.g. in syngo image rendering in the backend)

# Parallelism
## How to use Tasks for Parallelism (Task.Parallel, etc.)
* Show how this reduces number of threads and memory usage
* Show how this simplifies code
* Talk about data partioning in general

### Debugging tools in Visual Studio for TPL
* Parallel Stacks (Threads vs. Tasks)

### A little Parallelism Theory
https://en.wikipedia.org/wiki/Amdahl%27s_law
Bring example from bográcsozás to make it easily understandable
Show some code (stopwatch with internally having thread.sleep / task.delay?)

# Asynchronicity
## SynchronizationContext as an abstraction over scheduling
* Show that UI (WPF / WinForms) is *logically* a "synchronization context".
* Cross-thread access issues in WPF + discuss briefly why WPF was implemented this way

Starting point
```csharp
    /// <summary>
    /// Interaction logic for MainWindow.xaml
    /// </summary>
    public partial class MainWindow : Window
    {
        public MainWindow()
        {
            InitializeComponent();
        }

        private void Button_Click(object sender, RoutedEventArgs e)
        {
            Output.Text = $"Value of Pi is {Math.PI:0.0000000}";
        }
    }
```

Why doesn't this work?

```csharp
    /// <summary>
    /// Interaction logic for MainWindow.xaml
    /// </summary>
    public partial class MainWindow : Window
    {
        public MainWindow()
        {
            InitializeComponent();
        }

        private void Button_Click(object sender, RoutedEventArgs e)
        {
            Output.Text = "Value of Pi is: ";
            for(int i = 0; i < 10; i++)
            {
                char currentDigit = PiCalculator.GetPiDigitAsync(i).Result;
                Output.Text += currentDigit;
            }
        }
    }
```

Goal:
```csharp
    /// <summary>
    /// Interaction logic for MainWindow.xaml
    /// </summary>
    public partial class MainWindow : Window
    {
        public MainWindow()
        {
            InitializeComponent();
        }

        private async void Button_Click(object sender, RoutedEventArgs e)
        {
            Output.Text = "Value of Pi is: ";
            for(int i = 0; i < 10; i++)
            {
                char currentDigit = await PiCalculator.GetPiDigitAsync(i);
                Output.Text += currentDigit;
            }
        }
    }
```

### What happens in the background?

Why doesn't this work?

(Continuation is not scheduled in the same `SynchronizationContext`)
```csharp
    /// <summary>
    /// Interaction logic for MainWindow.xaml
    /// </summary>
    public partial class MainWindow : Window
    {
        public MainWindow()
        {
            InitializeComponent();
        }

        private void Button_Click(object sender, RoutedEventArgs e)
        {
            Output.Text = "Value of Pi is: ";
            for(int i = 0; i < 10; i++)
            {
                PiCalculator.GetPiDigitAsync(i)
                            .ContinueWith(t =>
                            {
                                Output.Text += t.Result;
                            });
            }
        }
    }
```

How to fix it?
And why is this not sufficient? (order of continuations is random at the scheduler!)
```csharp
    /// <summary>
    /// Interaction logic for MainWindow.xaml
    /// </summary>
    public partial class MainWindow : Window
    {
        public MainWindow()
        {
            InitializeComponent();
        }

        private void Button_Click(object sender, RoutedEventArgs e)
        {
            Output.Text = "Value of Pi is: ";
            for(int i = 0; i < 10; i++)
            {
                PiCalculator.GetPiDigitAsync(i)
                            .ContinueWith(t =>
                            {
                                Output.Text += t.Result;
                            }, TaskScheduler.FromCurrentSynchronizationContext());
            }
        }
    }
```

How to fix it?

Go to first implementation and explain what really happens here :) Threads, continuations, etc. Maybe look at disassembler?

### Parallel.For, ForEach
Take an example based on Enumerable.Range

Limitations
* blocking wait on the API!
* still requires synchronization, ideally don't use shared variables

### Task.WhenAll, Task.WhenAny
Rewrite `for (int i = 0; i < allTasks.Count; i++) Task.Wait()`

### IAsyncDisposable, IAsyncEnumerable
Rewrite `PiCalculator`

# Thread Safety, Concurrent Access of shared variables, Ensuring Data Integrity, Data Safety
* ideally: no shared variable usage, see functional style programming
* 2nd best option: use thread-safe constructs, see (System.Collections.Concurrent)
* 3rd best option: manually synchronize not thread-safe variables

* Logical synchronization vs. Type Thread-Safety (e.g. Concurrent collections need to be used wisely, they don't solve all *logical* concurrency problems, like unintended data overwrite from multiple writers)
* Mention some other "bigger picture" paradigms like
    * asynchronicity in cloud-based applications
    * Optimistic Concurrency Control

## Misc. Good to know stuff & Some Corner cases
* SyncContext is null in certain scenarios (e.g. Unit Test)
* Task.Wait or Task.Result can deadlock (also when mixed with await!)
* `async static Task<int> Main`
* how to offer async-based API from a non-async code (e.g. `Process.Start`, etc.)?

## Next steps
* Synchronization primitives (Mutex, IPC synchronization, etc.)

# Recommended Books
* [https://learn.microsoft.com/en-us/dotnet/csharp/programming-guide/concepts/async/task-asynchronous-programming-model] A good and readable introduction to the Task-based Async Pattern and its advantages over previous async solutions in .NET
* [Potential Pitfalls in Data and Task Parallelism](https://docs.microsoft.com/en-us/dotnet/standard/parallel-programming/potential-pitfalls-in-data-and-task-parallelism): A good starting point to help avoid typical mistakes
* [Stephen Cleary's blog](https://blog.stephencleary.com/2013/11/there-is-no-thread.html): Lots of good articles on asynchronicity and parallelism in .NET. Start with "There is no thread" ;-)
* [Patterns of Parallel Programming](https://www.microsoft.com/download/details.aspx?id=19222): Stephen Toub's great didactic introduction of how one would implement the async/parallel constructs on their own
* [CLR via C#](https://www.oreilly.com/library/view/clr-via-c/9780735668737/): A generally great book for understanding the internals of .NET