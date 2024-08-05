고급 식당에 비유. 각 식당은 프로세스(실행되는 프로그램), 식당마다 배치된 로봇 직원은 쓰레드, 그리고 로봇 직원이 움직이기 필요해 필요한 영혼이 바로  CPU의 코어이다. 여러 쓰레드를 번갈아가며 실행해주면 거의 동시에 모든 프로세스가 작동하는 것처럼 보일 것이다. 이것이 바로 멀티 쓰레딩의 기초이다.   

### Thread, ThreadPool, Task 예제
```cs
using System;
using System.Threading;
using System.Threading.Tasks;

namespace ServerCore
{
    class Program
    {
        static void MainThread(object state)
        {
            for(int i = 0; i < 5; i++)
                Console.WriteLine("Hello Thread!");
        }
        static void Main(string[] args)
        {
            ThreadPool.SetMinThreads(1, 1);
            ThreadPool.SetMaxThreads(5, 5);

            for (int i = 0; i < 5; i++)
            {
                Task t = new Task(() => { while (true) { } }, TaskCreationOptions.LongRunning);
                t.Start();
            }

            //for (int i = 0; i < 4; i++)
            //{
            //    threadpool.queueuserworkitem((obj) => { while (true) { } });
            //}

            ThreadPool.QueueUserWorkItem(MainThread);

            //Thread t = new Thread(MainThread);
            //t.Name = "Test Thread";
            //t.IsBackground = true;
            //t.Start();

            //Console.WriteLine("Waiting for Thread!");

            //t.Join();
            //Console.WriteLine("Hello World");
            while(true)
            {

            }
        }
    }
}
```

### Thread
- 평소 클래스 인스턴스를 생성하던 것처럼 생성한다.
- Name : IDE 상에서 표시되는 스레드의 이름을 설정할 수 있다.
- IsBackground : 디폴트는 false이다(C++과 다른 점이라고 한다).  백그라운드나 포그라운드에서 돌아갈 지 결정하는 Bool 프로퍼티인데, true 일 경우 해당 스레드는 백그라운드 스레드가 되어, 메인 스레드의 작업이 완료되면 백그라운드 스레드의 작업은 완료 여부와 상관 없이 종료된다.
- Start() : 스레드에 할당된 작업을 시작함
- Join() : 메인 스레드의 입장에서, 해당 스레드의 작업이 끝날 때까지 기다렸다가 이후 작업을 실행함.  

#### ThreadPool  
- Thread는 정직원의 느낌이라면, ThreadPool은 인력사무소, 단기 알바 고용하는 느낌
- 일일히 Thread를 생성하여 작업을 맡기고 Thread를 번갈아가며 실행해주는 것은 비효율적
- ObjectPool처럼, ThreadPool을 사용하는 것이 효과적이다. 스레드마다 어떤 작업을 누가 실행할 지 세세하게 정해줄 필요 없이, 필요한 작업을 맡기면 ThreadPool이 알아서 진행한다.
- SetMinThreads,SetMaxThreads로 최소, 최대 스레드 수를 설정할 수 있다.
- **주의! 너무 긴 작업을 여러 개 맡기면 ThreadPool이 먹통이 될 수 있다.**   

###  Task
-  직원 고용 X. 직원이 할 일을 정한다.
-  비동기 작업!
-  위 예제 코드처럼, Task를 생성할 때 TaskCreationOptions.LongRunning을 통해 소요시간이 오래 걸릴 작업을 설정할 수 있다. 이러면 ThreadPool만을 사용했을 때처럼 먹통이 되지 않고 작업이 잘 수행된다.