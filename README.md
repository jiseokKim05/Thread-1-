# Thread-1-
# Mutex란?

- Mutual exclusion의 줄임말이다. lock을 건다는 말과 같다.
- 스레드간에 동기화와 여러 스레드에서 같은 메모리에 쓸 때 공유되는 데이터를 보호한다.
- mutex 변수는 공유 데이터를 접근하는 것을 보호하기 위해 lock처럼 사용된다.
- mutex변수를 공유하는 스레드 중 하나의 스레드만이 mutex 변수에 lock을 걸 수 있다.
- 공유된 변수를 번갈아가며 사용할 때 사용된다 (race condition 방지)

# Mutex의 사용 예시

![image.png](attachment:a4b26516-9c8d-4369-aca5-7c0a05b16ed8:image.png)

위 그림과 같이 은행 데이터베이스에 shyeon은 1000원, gildong은 1500원이 있다고 하자. 이때 A에서 shyeon에게 500원을 송금하고자하고 동시에 B에서 700원을 송금하고자 함. 이때 무슨 일이 일어날까? 각각 A, B에서는 이런 일이 발생할 것이다.

- 은행 DB로부터 shyeon의 계좌 정보를 변수에 저장한다.
- 그 변수에 500원을 더한다.
- 은행 DB에 계산 결과값을 저장한다.

A, B에서 위의 일이 동시에 발생한다면 A, B 모두 은행 계좌로부터 shyeon의 돈은 1000원이 있다고 읽어오고 A는 1000원에 500원을 더한 1500원을 은행 DB에 보낼 것이고 B는 700원을 더한 1700원을 DB에 보낼 것이다. 이때 더 늦게 도착하는 값으로 DB에 있는 shyeon의 잔고가 갱신될 것이다. 이런 상황을 race condition이라고 말한다. 어떤게 먼저 도착하냐에 따라 값이 달라지는 상황이다. 이런 일을 방지하려면 어떻게 해야할까? A나 B가 순서를 정하여 DB에 접근하면 된다. 이런 순서를 정하기 위해 mutex가 사용된다.

---

# mutex 관련 함수

## 1. pthread_mutex_init

```cpp
int pthread_mutex_init(ptrhead_mutex_t* mutex, const pthread_mutexatt_t *attr);
```

mutex변수를 초기화하는데 사용하는 변수임. 

mutex변수를 선언하여 첫 번째 인자에 주소값을 넘겨 초기화 할 수 있음. attr인자는 NULL로 하면 가본 속성으로 생성할 수 있음. 리눅스에서는 기본값인 NULL값을 사용함. mutex변수를 초기화하는 또다른 방법으로는 다음과 같이 선언하면 됨.

```cpp
pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;
```

위의 코드처럼 mutex변수에 직접 대입하는 방법임. 이 방법은 attr인자를 사용할 수 없다는 단점이 있지만 리눅스에서는 고려하지 않음.

## 2. pthread_mutex_destroy

```cpp
int pthread_mutex_destroy(pthread_mutex_t *mutex);
```

사용이 끝난 mutex 변수를 해제하는 함수. 

malloc으로 동적으로 선언한 mutex 변수는 free()를 호출하기 전에 꼭pthread_mutex_destroy()를 먼저 호출해야함.

## 3. pthread_mutex_lock

```cpp
int pthread_mutex_lock(pthread_mutex_t *mutex);
```

mutex 변수에 lock을 거는 함수임.

두 스레드의 함수에서 같은 mutex변수에 대해 각각 pthread_mutex_lock을 호출할 경우 두 스레드 중 하나의 스레드만 mutex 변수에 대해 lock을 얻고 다른 스레드에서는 pthread_mutex_lock에서 기다리고 있음. 이때 mutex를 얻은 스레드에서 pthread_mutex_unlock()을 호출할 경우 다른 스레드에서 mutex변수에 lock을 걸 수 있음.

---

# 예제

아래 코드는 두 스레드에서 공유하는 하나의 변수에 번갈아가며 출력하는 예제

### 1번 코드

```cpp
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <pthread.h>

pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;
int value;

void *func1(void *arg) { //1번 스레드가 실행할 함수
	for(int  i = 0; i < 5; i++) {
		pthread_mutex_lock(&mutex); //mutex 변수 lock(다른 변수에서 접근 불가능)
		printf("thread1 : %d\n", value); //변수 출력
		value++; //변수 증가
		pthread_mutex_unlock(&mutex); //mutex 변수 unlock(다른 변수에서도 접근 가능)
		sleep(1);
	}

	pthread_exit(NULL); //스레드 함수 종료
}

void *func2(void *arg) { //2번 스레드가 실행할 함수
	for(int i = 0; i < 5; i++) { 
		pthread_mutex_lock(&mutex); //mutex 변수 lock(다른 변수에서 접근 불가능)
		printf("thread2 : %d\n", value); //변수 출력
		value++; //변수 증가
		pthread_mutex_unlock(&mutex); //mutex 변수 unlock(다른 변수에서 접근 가능)
		sleep(1);
	}

	pthread_exit(NULL); //스레드 함수 종료
}

int main() {
	pthread_t tid1, tid2; //스레드 변수 선언

	value = 0; //공유 변수 초기화

	if(pthread_create(&tid1, NULL, func1, NULL) != 0) { //1번 스레드 생성
		fprintf(stderr, "pthread create error\n");
		exit(1);
	}

	if(pthread_create(&tid2, NULL, func2, NULL) != 0) { //2번 스레드 생성
		fprintf(stderr, "pthread create error\n");
		exit(1);
	}

	if(pthread_join(tid1, NULL) != 0) { //1번 스레드 종료 후 리소스 회수
		fprintf(stderr, "pthread join error\n");
		exit(1);
	}

	if(pthread_join(tid2, NULL) != 0) { //2번 스레드 종료 후 리소스 회수
		fprintf(stderr, "pthread join error\n");
		exit(1);
	}

	pthread_mutex_destroy(&mutex); //mutex 변수 해제

	exit(0);
}
```

![image.png](attachment:0e8bbafc-f15d-4166-bd59-17f9cc3fa9ec:b04f37fd-748e-43d0-9258-241670328714.png)

### 2번 코드

```cpp
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <pthread.h>

pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;
int value;

void *func1(void *arg) {
    //1번 스레드가 실행할 함수
    for(int i = 0; i < 5; i++) {
        //pthread_mutex_lock(&mutex); //mutex 변수 lock(다른 변수에서 접근 불가능)
        printf("thread1 : %d\n", value); //변수 출력
        value++; //변수 증가
        //pthread_mutex_unlock(&mutex); //mutex 변수 unlock(다른 변수에서도 접근 가능)
        sleep(1);
    }
    pthread_exit(NULL); //스레드 함수 종료
}

void *func2(void *arg) {
    //2번 스레드가 실행할 함수
    for(int i = 0; i < 5; i++) {
        //pthread_mutex_lock(&mutex); //mutex 변수 lock(다른 변수에서 접근 불가능)
        printf("thread2 : %d\n", value); //변수 출력
        value++; //변수 증가
        //pthread_mutex_unlock(&mutex); //mutex 변수 unlock(다른 변수에서 접근 가능)
        sleep(1);
    }
    pthread_exit(NULL); //스레드 함수 종료
}

int main() {
    pthread_t tid1, tid2; //스레드 변수 선언
    value = 0; //공유 변수 초기화

    if(pthread_create(&tid1, NULL, func1, NULL) != 0) { //1번 스레드 생성
        fprintf(stderr, "pthread create error\n");
        exit(1);
    }

    if(pthread_create(&tid2, NULL, func2, NULL) != 0) { //2번 스레드 생성
        fprintf(stderr, "pthread create error\n");
        exit(1);
    }

    if(pthread_join(tid1, NULL) != 0) { //1번 스레드 종료 후 리소스 회수
        fprintf(stderr, "pthread join error\n");
        exit(1);
    }

    if(pthread_join(tid2, NULL) != 0) { //2번 스레드 종료 후 리소스 회수
        fprintf(stderr, "pthread join error\n");
        exit(1);
    }

    pthread_mutex_destroy(&mutex); //mutex 변수 해제
    exit(0);
}
```

첫 번째 사진은 mutex lock을 사용한 결과이고 두 번째 사진은 pthread_mutex_lock(), pthread_mutex_unlock()부분을 주석처리하고 실행한 결과임. 

실행 결과 1번 사진과 같이 1번 스레드와 2번 스레드가 번갈아가며 변수를 증가하며 출력하는 것을 볼 수 있음.

2번 사진은 공유된 변수에 동시에 접근하여 같이 출력되는 것을 알 수 있음. 

이렇게 여러 스레드가 공유 변수에 접근하고자할 때 race condition을 방지하기 위해 mutex를 사용할 수 있음.

pthread 라이브러리를 사용할 때는 컴파일할 때 -lpthread를 포함해줘야함. 그렇지 않으면 에러가 발생하고 컴파일이 되지 않음.
