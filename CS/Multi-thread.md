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

## 메모리 배리어
```cs
using System;
using System.Threading;
using System.Threading.Tasks;

namespace ServerCore
{
    class Program
    {
        static int x = 0;
        static int y = 0;
        static int r1 = 0;
        static int r2 = 0;

        static void Thread1()
        {
            y = 1; // Store y
            r1 = x; // Load x
        }

        static void Thread2()
        {
            x = 1; // Store x
            r2 = y; //Load y
        }
        static void Main(string[] args)
        {
            int count = 0;
            while(true)
            {
                count++;
                x = y = r1 = r2 = 0;

                Task t1 = new Task(Thread1);
                Task t2 = new Task(Thread2);
                t1.Start();
                t2.Start();

                Task.WaitAll(t1, t2);

                if (r1 == 0 && r2 == 0)
                    break;
            }
            Console.WriteLine($"{count}번만에 빠져나옴");
        }
    }
}

```
위 예제를 보자. 메인 메서드의 무한 루프를 빠져나오려면, r1과 r2 모두 0이 되어야 한다. 하지만 상식적으로, 두 스레드의 작업을 보면 알 수 있듯이 두 결과값이 동시에 0인 경우는 발생할 수 없다는 것을 알 수 있다. 하지만 놀랍게도 위 코드의 실행 결과화면을 보자.
![[Pasted image 20240807135308.png]]
횟수는 늘 달라지지만, 어이없게도 늘 무한루프를 빠져나오는 현상이 발견된다. 왜 이럴까?

### 하드웨어 최적화
저번에 배웠던 컴파일러 최적화처럼, 하드웨어도 지 맘대로 코드를 재배치하며 최적화를 하곤 한다. 예를 들어 각 스레드에서 값을 Store하고 Load하는 부분이, 컴퓨터 입장에서 보면 순서가 바뀌어도 상관이 없다고 봐서, 멋대로 코드를 재배치할 수 도 있다는 것이다. 하지만 위 코드에서 store와 load하는 부분을 바꿔버리면,  결과값이 둘 다 0이 되는 경우가 발생하게 되어, 무한루프에서 빠져나올 수 있게 되는 것이다. 하하!  

그렇다면 이러한 의도치 않은 최적화를 막기 위해서는 어떤 방법이 있을까?

### Thread.MemoryBarrier()
```cs
using System;
using System.Threading;
using System.Threading.Tasks;

namespace ServerCore
{
    class Program
    {
        static int x = 0;
        static int y = 0;
        static int r1 = 0;
        static int r2 = 0;

        static void Thread1()
        {
            y = 1; // Store y
            Thread.MemoryBarrier();
            r1 = x; // Load x
        }

        static void Thread2()
        {
            x = 1; // Store x
            Thread.MemoryBarrier();
            r2 = y; //Load y
        }
        static void Main(string[] args)
        {
            int count = 0;
            while(true)
            {
                count++;
                x = y = r1 = r2 = 0;

                Task t1 = new Task(Thread1);
                Task t2 = new Task(Thread2);
                t1.Start();
                t2.Start();

                Task.WaitAll(t1, t2);

                if (r1 == 0 && r2 == 0)
                    break;
            }
            Console.WriteLine($"{count}번만에 빠져나옴");
        }
    }
}

```
앞서 소개된 코드에서 store와 load하는 부분 사이에 코드 한 줄만 추가한 것이다. 그런데 결과는 놀랍게도 원래 의도처럼 무한루프에서 빠져나오지 않게 변한 것을 볼 수 있다. 

메모리 배리어의 역할은 크게 2가지가 있다.
- 코드 재배치 억제 : 말 그대로
- 가시성 : (맞는건지 모르겠다) Store 하면 바로 메인 메모리에 갖다 쓰고, Load하면 캐시가 아니라 메인 메모리에서 가져와서 혼선을 막는..그런걸로 일단 이해함

위 예제에서는 코드 재배치를 억제하여 정상적으로 작동할 수 있게 되었다. 메모리 배리어는 종류도 3가지가 있다. 이름 그대로의 역할을 한다. 참 쉽죠잉?
- Full Memory Barrier
- Store Memory Barrier
- Load Memory Barrier


## Interlocked
```cs
using System;
using System.Threading;
using System.Threading.Tasks;

namespace ServerCore
{
    class Program
    {
        static int number = 0;
        
        static void Thread1()
        {
            for (int i = 0; i < 1000000; i++)
                number++
        }

        static void Thread2()
        {
            for (int i = 0; i < 1000000; i++)
                number--;
        }
        static void Main(string[] args)
        {
            Task t1 = new Task(Thread1);
            Task t2 = new Task(Thread2);
            t1.Start();
            t2.Start();

            Task.WaitAll(t1, t2);

            Console.WriteLine(number);
        }
    }
}

```
같은 횟수만큼 더하고, 같은 횟수만큼 빼는 작업이므로 우리가 하는 수학적 상식에 의하면 결과는 0이어야 한다. 하지만?
![[Pasted image 20240808145940.png]]
또 멀티 쓰레드 환경에서는 우리의 상식이 통하지 않는 모습을 볼 수 있다. number를 `volatile` 로 선언해도 소용없더라. 이는 경합 조건 때문에 일어나는 현상이다. 

### 경합 조건 (Race Condition)
고급 식당에 비유해보자. 앞서 우리는 직원들(스레드)이 주문 현황표(메인 메모리)로부터 주문을 확인하도록 하여 변경사항에 잘 대응할 수 있는 방법을 알아보았다. 2번 테이블에서 콜라 한 병을 주문했다. 이때 새로 들어온 열정 넘치는 신입 직원 여러명이 동시에 2번 테이블로 콜라를 배달하려고 한다. 2번 테이블 손님은 콜라 한 병만 주문했는데 콜라 3병을 받아버린다. 이게 경합조건이다.

한편으로는 이렇게 생각할 수도 있다. 단순히 숫자를 더하고 빼는 작업인데, 그리고 같은 횟수만큼 진행할 거, 순서가 좀 변하고 엎치락뒤치락 해도 결과는 0이 나와야 하는거 아니냐! 하지만 number++, number--의 과정은 생각보다 단순하지 않더라. 해당 작업에 break point를 걸고 디스어셈블리 창을 켜보자.
![[Pasted image 20240808152626.png]]
우리의 생각처럼 한방에 이루어지는 작업이 아니더라. 이걸 이해하기 쉽게 알아볼 수 있는 언어로 써보면 다음과 같은 작업이다.
```cs
int temp = number;
temp += 1;
number = temp;

int temp = number;
temp -= 1;
number = temp;
```
0부터 시작했을 때, 스레드 1은  `number`에 1을 넣으려 할 것이고, 스레드 2는 -1을 넣으려 할 것이다. 그러나 누가 먼저 접근할지는 모른다. 이런 일이 계속돼다 보니 위에서 나온 참사가 발생하는 것이다. 이런 일을 방지하기 위해서는 number++, number--의 작업, 즉 값을 메모리에서 가져오고, 변경하고, 다시 쓰는 작업이, 동시에 일어나야 한다. **<font color="#ffc000">원자적으로</font>** 일어나야 한다는 것이다. 이러한 특성을 <font color="#ffc000">원자성(Atomic)</font>이라고 한다.
### Interlocked
자 그럼 저 작업이 원자적으로 일어나게 하려면 어떻게 할까?
```cs
using System;
using System.Threading;
using System.Threading.Tasks;

namespace ServerCore
{
    class Program
    {
        static int number = 0;
        
        static void Thread1()
        {
            for (int i = 0; i < 1000000; i++)
                Interlocked.Increment(ref number);
        }

        static void Thread2()
        {
            for (int i = 0; i < 1000000; i++)
                Interlocked.Decrement(ref number);
        }
        static void Main(string[] args)
        {
            Task t1 = new Task(Thread1);
            Task t2 = new Task(Thread2);
            t1.Start();
            t2.Start();

            Task.WaitAll(t1, t2);

            Console.WriteLine(number);
        }
    }
}

```
`Interlocked` 클래스를 활용하는 방법이 있다. 
![[Pasted image 20240808153942.png]]
정상적으로 0이 출력되는 모습을 볼 수 있다. 그리고 Interlocked를 사용한 순간 관련된 변수는 자동으로 volatile 과 비스무리하게 처리되기 때문에 굳이 volitile 키워드를 같이 사용할 필요도 없다. 그렇다고 언제 어디서나 Interlocked를 자유자재로 사용하면 좋겠구나! 라고 하면 안된다. Interlocked 작업은, All or nothing. 하나의 작업만 진행한다. 즉 성능적으로 손해를 보게 된다. 시기적절하게 사용하도록 하자.   

## Lock 기초
앞선 시간에 Interlocked를 사용해서 경합조건을 방지하는 방법을 알아보았다. 하지만 Interlocked로만 구현하기에는 어려운 기능을 atomic하게 구현해야 할 때도 있는데, 이럴 때 사용할 수 있는 방법을 알아보자.  
### Monitor.Enter & Exit
```cs
using System;
using System.Threading;
using System.Threading.Tasks;

namespace ServerCore
{
    class Program
    {
        static int number = 0;
        static object _obj = new object();
        
        static void Thread1()
        {
            for (int i = 0; i < 1000000; i++)
            {
                // 상호배제 Mutual Exclusive
                Monitor.Enter(_obj); //문을 잠그는 행위
                number++;
                Monitor.Exit(_obj); // 잠금을 풀어준다
            }
                
        }

        static void Thread2()
        {
            for (int i = 0; i < 1000000; i++)
            {
                Monitor.Enter(_obj);
                number--;
                Monitor.Exit(_obj);
            }
                
        }
        static void Main(string[] args)
        {
            Task t1 = new Task(Thread1);
            Task t2 = new Task(Thread2);
            t1.Start();
            t2.Start();

            Task.WaitAll(t1, t2);

            Console.WriteLine(number);
        }
    }
}

```  
먼저 스태틱 변수로 오브젝트를 선언해준다. 이 오브젝트는 앞으로 '자물쇠' 같은 역할을 한다. `Moniter.Enter`는 문을 잠그고, `Monitor.Exit` 는 잠금을 해제하는 역할을 한다. 이렇게 둘의 사이에 있는 코드들은 상호 배제 하게 되어, `Interlocked`를 사용해서 `number` 변수가 변화되는 부분을 atomic하게 했던 것 같은 효과를 볼 수 있다. 
다만, 반드시 Enter와 Exit가 짝으로 이뤄져야 한다. Enter는 했는데 예외가 발생하거나, 실수로 중간에 `return`을 때려버린다거나 등으로, Exit로 잠금을 해제해주지 않으면, 상호 배제 상태인 다른 쪽 Enter&Exit 부분이 <font color="#ffc000">Deadlock</font> 상태가 되어 영영 실행되지 않는다. 마치 화장실에 똥싸러 가서 문을 잠그고, 다시 안 열어줘서 다른 사람이 똥을 싸러 가지 못하는 것과 같은 이치다. 그렇다면 이런 문제를 어떻게 예방할 수 있을까?

```cs
using System;
using System.Threading;
using System.Threading.Tasks;

namespace ServerCore
{
    class Program
    {
        static int number = 0;
        static object _obj = new object();
        
        static void Thread1()
        {
            for (int i = 0; i < 1000000; i++)
            {
                try
                {
                    // 상호배제 Mutual Exclusive
                    Monitor.Enter(_obj); //문을 잠그는 행위
                    number++;
                }
                finally
                {
                    Monitor.Exit(_obj); // 잠금을 풀어준다
                }
                
            }
                
        }

        static void Thread2()
        {
            for (int i = 0; i < 1000000; i++)
            {
                Monitor.Enter(_obj);
                number--;
                Monitor.Exit(_obj);
            }
                
        }
        static void Main(string[] args)
        {
            Task t1 = new Task(Thread1);
            Task t2 = new Task(Thread2);
            t1.Start();
            t2.Start();

            Task.WaitAll(t1, t2);

            Console.WriteLine(number);
        }
    }
}

```
위처럼 `try-finally`를 이용하면 예외가 발생해도, `return`을 때려도 finally 문 안의 코드는 반드시 실행되기 때문에 `Monitor.Exit`를 무슨 일이 있어도 실행할 수 있다. 하지만 늘 이런 식으로 코드를 짜기에는 번거롭다. 더 쉬운 방법을 알아보자.

### Lock
```cs
using System;
using System.Threading;
using System.Threading.Tasks;

namespace ServerCore
{
    class Program
    {
        static int number = 0;
        static object _obj = new object();
        
        static void Thread1()
        {
            for (int i = 0; i < 1000000; i++)
            {
                lock(_obj)
                {
                    number++;
                }
                
            }
                
        }

        static void Thread2()
        {
            for (int i = 0; i < 1000000; i++)
            {
                lock (_obj)
                {
                    number--;
                }
            }
                
        }
        static void Main(string[] args)
        {
            Task t1 = new Task(Thread1);
            Task t2 = new Task(Thread2);
            t1.Start();
            t2.Start();

            Task.WaitAll(t1, t2);

            Console.WriteLine(number);
        }
    }
}

```  
앞서 나온 내용을 쉽게 사용할 수 있는 것 이 `Lock`이다. 이 기능을 적극적으로 활용하도록 하자. 지가 알아서 잠그고 풀고 한다.  

## Deadlock

이전 시간에 `Moniter.Enter &Exit`가 쌍으로 일어나지 않으면 상호 배제된 다른 쪽 코드가 `Deadlock`이 걸리는 예제를 봤다. 하지만 이런 경우는 1차원적인 예제로, 프로그래머가 굉장히 멍청한 경우라고 볼 수 있다. 실제 발생되는 Deadlock은 어떻게 일어나는 것인지 알아보자.
![[Pasted image 20240827132431.png]]
자물쇠가 여러개일 때, 그리고 한 스레드가 두 자물쇠 모두에 접근해야 하는데, 위처럼 다른 스레드가 자물쇠를 하나씩 물고 있어 버리면 둘 다 Deadlock 상태가 되어버린다. 

```cs
using System;
using System.Threading;
using System.Threading.Tasks;

namespace ServerCore
{
    class SessionManager
    {
        static object _lock = new object();

        public static void TestSession()
        {
            lock (_lock)
            {
                
            }
        }

        public static void Test()
        {
            lock (_lock)
            {
                UserManager.TestUser();
            }
        }
    }

    class UserManager
    {
        static object _lock = new object();

        public static void Test()
        {
            lock (_lock)
            {
                SessionManager.TestSession();
            }
        }

        public static void TestUser()
        {
            lock (_lock)
            {
                
            }
        }
    }

    class Program
    {
        static int number = 0;
        static object _obj = new object();
        
        static void Thread1()
        {
            for (int i = 0; i < 10000; i++)
            {
                SessionManager.Test();
                
            }
                
        }

        static void Thread2()
        {
            for (int i = 0; i < 10000; i++)
            {
                UserManager.Test();
            }
                
        }
        static void Main(string[] args)
        {
            Task t1 = new Task(Thread1);
            Task t2 = new Task(Thread2);
            t1.Start();
            t2.Start();

            Task.WaitAll(t1, t2);

            Console.WriteLine(number);
        }
    }
}

```
위 예제에서는 SessionManager, UserManager가 서로의 메서드를 호출하는데, 다 lock이 걸려있다. 물론 코드대로 딱딱 실행된다면 문제없이 종료되어야 하는데, 코드를 실행해보면 무한 루프에 빠져서 실행이 끝나지 않는 모습을 볼 수 있다. 실행 후 일시정지를 눌러 어디에서 걸려있는지 확인해보면,
![[Pasted image 20240827145056.png]]
스레드에서의 작업이 끝나지 않아 위처럼 하염없이 기다리고 있는 모습을 볼 수 있다. 각 작업자 스레드의 입장도 살펴보자.
![[Pasted image 20240827145144.png]]![[Pasted image 20240827145212.png]]
맨 처음 다이어그램과 유사하게, 서로 막 호출하다가 데드락 상태에 빠진 모습을 볼 수 있다. 이런 복잡한 데드락은 저번과 같은 명확한 해결책은 없다. 몇 가지 해결책이 있긴 한데 그 중 하나는 다음과 같다.  
### Monitor.TryEnter
 문을 계속 두드리는 데 열리지 않는다면, 상식적으로 시간이 좀 지난 후에 포기할 수도 있지 않은가? 이 방법이 바로 `Monitor.TryEnter()`를 사용하는 방법이다.
 ![[Pasted image 20240827145912.png]]
 이런게 있긴 하지만, `Try-Catch`문과 비슷하게, 궁극적인 문제점을 해결하는게 아니다 보니 사용을 크게 추천하진 않는다. 애초에 데드락이 발생했다는 것은 락 구조에 문제가 있다는 것이므로, 락 구조를 바꾸는 식으로 문제를 해결하는 방향이 바람직하다. 사실 위 예제 코드의 데드락은 두 `Task` 를 동시에 실행하지 않고, 약간의 시간차를 주기만 하면 해결된다.
```cs
using System;
using System.Threading;
using System.Threading.Tasks;

namespace ServerCore
{
    class SessionManager
    {
        static object _lock = new object();

        public static void TestSession()
        {
            lock (_lock)
            {
                
            }
        }

        public static void Test()
        {
            lock (_lock)
            {
                UserManager.TestUser();
            }
        }
    }

    class UserManager
    {
        static object _lock = new object();

        public static void Test()
        {
            lock (_lock)
            {
                SessionManager.TestSession();
            }
        }

        public static void TestUser()
        {
            lock (_lock)
            {
                
            }
        }
    }

    class Program
    {
        static int number = 0;
        static object _obj = new object();
        
        static void Thread1()
        {
            for (int i = 0; i < 10000; i++)
            {
                SessionManager.Test();
                
            }
                
        }

        static void Thread2()
        {
            for (int i = 0; i < 10000; i++)
            {
                UserManager.Test();
            }
                
        }
        static void Main(string[] args)
        {
            Task t1 = new Task(Thread1);
            Task t2 = new Task(Thread2);
            t1.Start();

            Thread.Sleep(100);

            t2.Start();

            Task.WaitAll(t1, t2);

            Console.WriteLine(number);
        }
    }
}

```

## Lock 구현 이론

다시 식당 비유로 돌아가서 Lock 구현 이론을 비유적으로 알아보자.
화장실에 갔는데, 안에 사람이 있을 때 어떻게 할 수 있을까?
### 존버 메타 (SpinLock)
![[Pasted image 20240902230956.png]]
```cs
using System;
using System.Threading;
using System.Threading.Tasks;

namespace ServerCore
{
    class SpinLock
    {
        volatile int _locked = 0;

        public void Acquire()
        {
            while (true)
            {
                //int original = Interlocked.Exchange(ref _locked, 1);
                //if (original == 0)
                //    break;

                //CAS Compare-And-Swap
                int expected = 0;
                int desired = 1;
                if (Interlocked.CompareExchange(ref _locked, desired, expected) == expected)
                    break;
            }
        }

        public void Release()
        {
            _locked = 0;
        }
    }

    class Program
    {
        static int _num = 0;
        static SpinLock _lock = new SpinLock();

        static void Thread_1()
        {
            for (int i = 0; i < 100000; i++)
            {
                _lock.Acquire();
                _num++;
                _lock.Release();
            }
        }

        static void Thread_2()
        {
            for (int i = 0; i < 100000; i++)
            {
                _lock.Acquire();
                _num--;
                _lock.Release();
            }
        }

        static void Main(string[] args)
        {
            Task t1 = new Task(Thread_1);
            Task t2 = new Task(Thread_2);
            t1.Start();
            t2.Start();

            Task.WaitAll(t1, t2);

            Console.WriteLine(_num);
        }
    }
}

```
뽀인뜨 :
- Interlocked.Exchange & Interlocked.CompareExchange : 후자가 일반적으로 많이  쓰임
-  변수가 저장되는 곳 - 힙인가 스택인가. 스택이면 세이프 존. 데인저 존을 보는 눈을 기르자
### 랜덤 메타 (ContextSwitching)
![[Pasted image 20240902231024.png]]

### 갑질 메타 (AutoResetEvent)
![[Pasted image 20240902231032.png]]