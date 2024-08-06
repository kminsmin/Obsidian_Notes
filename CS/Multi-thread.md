고급 식당에 비유. 각 식당은 프로세스(실행되는 프로그램), 식당마다 배치된 로봇 직원은 쓰레드, 그리고 로봇 직원이 움직이기 필요해 필요한 영혼이 바로  CPU의 코어이다. 여러 쓰레드를 번갈아가며 실행해주면 거의 동시에 모든 프로세스가 작동하는 것처럼 보일 것이다. 이것이 바로 멀티 쓰레딩의 기초이다.   

## Thread, ThreadPool, Task
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



## 컴파일러 최적화  

멀티 쓰레딩을 이용해 프로그래밍을 하다 보면, 간혹 디버그 모드에서는 잘 작동하는데 릴리스 모드에서는 제대로 작동하지 않을 때가 있다. 이는 디버그 모드와는 다르게 릴리스 모드에서는 컴퓨터가 코드를 읽어나갈 때 어셈블리어 상에서 지멋대로 최적화를 진행하기 때문이다. 이를 컴파일러 최적화라고 한다.   
```cs
using System;
using System.Threading;
using System.Threading.Tasks;

namespace ServerCore
{
    class Program
    {
        volatile static bool _stop = false;
        static void ThreadMain()
        {
            Console.WriteLine("쓰레드 시작!");

            while(_stop == false)
            {
                // 누군가가 stop 신호를 해주기를 기다린다
            }

            Console.WriteLine("쓰레드 종료!");

        }
        static void Main(string[] args)
        {
            Task t = new Task(ThreadMain);
            t.Start();

            Thread.Sleep(1000); //밀리초 단위.

            _stop = true;

            Console.WriteLine("Stop 호출");
            Console.WriteLine("종료 대기중");
            t.Wait();//Task 가 끝나기를 기다림. Thread.Join()이랑 같은 기능
            Console.WriteLine("종료 성공");
        }
    }
}

```
위 코드를 보자. ThreadMain 메서드 내에서는 stop 변수가 변경되지 않는다는 것을 컴퓨터가 알아서, 이걸 본래 의도대로 해석하지 않고 아래 예시처럼 이상하게 최적화 해버린다. 

```cs
while(_stop == false)
{
    
}
//아래와 같이 멋대로 최적화해버린다
if(_stop == false)
{
	while(true)
	{
	
	}
}
```  
위 코드처럼 최적화되면 본래 코드와는 다르게 stop 변수가 수정되어도 무한 루프에 갇혀 빠져나오지 못하는 경우가 발생할 수 있다. 이를 방지하기 위한 방법은 여러가지가 있는데, 그 중 하나는 stop 변수를 `volatile`로 선언하는 것이다. 

### 공식 문서 설명 :
컴파일러, 런타임 시스템 및 하드웨어는 성능상의 이유로 메모리 위치에 대한 읽기 및 쓰기를 다시 정렬할 수 있습니다. `volatile`로 선언된 필드는 특정 종류의 최적화에서 제외됩니다.

### volatile 
- 사전상 의미 : 휘발성 물질
-  여러 스레드에 의해 필드가 수정될 수 있음을 나타내는 키워드
- 전문가들은 사용을 추천하지 않음. 이런게 있구나 정도만 알고 넘어가자

## 캐시 이론
![[Pasted image 20240806233902.png]]  
위 그림처럼 스레드는 메모리의 어떤 값에 접근할 때, 특정 원칙에 따라 미리 데이터를 저장해 놓는데 이를 캐시라 한다. 다음과 같은 원칙에 따라 캐싱이 일어난다.  
- Temporal Locality : 방금 주문이 들어온 테이블에서 또 주문이 들어올 거야!
- Spacial Locality : 방금 주문이 들어온 테이블 주변에서 또 주문이 들어올 거야!

### 캐싱 예제
```cs
using System;
using System.Threading;
using System.Threading.Tasks;

namespace ServerCore
{
    class Program
    {
        static void Main(string[] args)
        {
            int[,] arr = new int[10000, 10000];

            {
                long now = DateTime.Now.Ticks;
                for (int y = 0; y < 10000; y++)
                    for (int x = 0; x < 10000; x++)
                        arr[y, x] = 1;
                long end = DateTime.Now.Ticks;
                Console.WriteLine($"(y,x) 순서 걸린 시간 {end - now}");
            }

            {
                long now = DateTime.Now.Ticks;
                for (int y = 0; y < 10000; y++)
                    for (int x = 0; x < 10000; x++)
                        arr[x, y] = 1;
                long end = DateTime.Now.Ticks;
                Console.WriteLine($"(y,x) 순서 걸린 시간 {end - now}");
            }
        }
    }
}

```
실행 결과 :
![[Pasted image 20240806234344.png]]
수학적으로 생각해 보면, 분명 같은 시간이 걸려야 맞다. 하지만 아무리 많이 테스트해봐도, 후자가 더 오래 걸린다. 왜일까?
배열들의 배열을 검사한다. 전자는 검사할 때, 한 배열의 내부 요소를 전부 검사한 후 다음 배열로 넘어간다. 후자의 경우, 배열 하나의 몇 번째 요소를 검사 후,  다른 배열로 넘어가서 같은 순서의 요소를 검사하는 방식이다. 즉, 캐싱이 제대로 이루어지는 환경이라면, 앞서 언급된 원칙에 따라 특정 데이터에 접근할 때 그 주변 데이터나 최신 데이터를 캐싱하기 때문에, 메인 메모리까지 접근하지 않아서 데이터에 접근하는 시간이 줄어든다.

후기 : 새롭다...어렵다...